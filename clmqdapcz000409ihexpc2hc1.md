---
title: "Building User Authentication with Username and Password using Supabase"
seoTitle: "Enhance Authentication with Supabase"
seoDescription: "Boost user login options using Supabase. Learn to add username & password login alongside email & password for enhanced authentication."
datePublished: Tue Sep 19 2023 13:45:12 GMT+0000 (Coordinated Universal Time)
cuid: clmqdapcz000409ihexpc2hc1
slug: building-user-authentication-with-username-and-password-using-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1695129002603/14cb9317-3fa5-45a8-81e5-7d108a27c7d5.png
tags: postgresql, authentication, web-development, supabase

---

**User authentication is essential for web application security and user management.** [**Supabase**](https://supabase.com) **is an open-source platform that makes it easy to build secure authentication systems. In this blog post, we will show you how to enhance user login options by enabling users to log in with their username and password or their email and password. You can check this example also on the** [**GitHub repo**](https://github.com/mansueli/supabase-username-login)**.**

## Prerequisites

Before we embark on this tutorial, it's crucial to have a foundational understanding of web development and Node.js. Familiarity with Next.js, a popular React framework, will also be advantageous. Additionally, you need Node.js and npm (Node Package Manager) installed on your system.

## Setting up the Project

To kickstart our journey, let's create a new Next.js project integrated with Supabase. Execute the following command to establish the initial project structure:

```bash
npx create-next-app -e with-supabase username-login
```

This command will generate a Next.js project preconfigured with Supabase support with default authentication and routes.

## Configuring Supabase

Before we start coding, let's set up Supabase for our project. You need a Supabase project to get the credentials you need. Make sure you have your Supabase URL and API keys ready. It's important to keep these credentials safe, such as by using environment variables. You can find more information on configuring Supabase in the [Supabase documentation](https://supabase.com/docs).

## Tweaking the Sign-In Route

Now, let's closely examine the code responsible for user sign-in. Navigate to the `username-login/app/auth/sign-in/route.ts` file in your project. We will replace the default code for user authentication as follows:

Replace:

```js
const { error } = await supabase.auth.signInWithPassword({
  email,
  password,
})
```

With:

```typescript
const { data } = await supabase.functions.invoke('sign-in', {
  body: {
    email,
    password,
  }
});

const { error } = await supabase.auth.setSession({
  access_token: data.access_token,
  refresh_token: data.refresh_token
});
```

This code leverages Supabase Edge Functions to handle user sign-in, enabling users to choose between username and password or email for login.

## Edge Function for Sign-In

Within the authentication workflow, this edge function plays a pivotal role in managing sign-in requests. This function offers users greater flexibility by enabling them to use their usernames as an alternative login option.

To set up the Edge Function for Sign-In, create a Deno script as follows:

```typescript
import { serve } from 'https://deno.land/std@0.192.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
import { corsHeaders } from '../_shared/cors.ts'

console.log(`Function "sign-in" is up and running!`)

const options =  {
  auth: {
    flowType: 'implicit',
    autoRefreshToken: false,
    persistSession: false,
    detectSessionInUrl: false
  }
};

serve(async (req: Request) => {
  // This is needed if you're planning to invoke your function from a browser.
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
      options
    )
    const { email, password } = await req.json()
    // Create a Supabase client with the Auth context of the logged-in user.
    const supabaseClient = createClient(
      // Supabase API URL - env var exported by default.
      Deno.env.get('SUPABASE_URL') ?? '',
      // Supabase API ANON KEY - env var exported by default.
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      options
    )
    const { data: profileData, error: userError } = await supabaseAdmin
      .from('profiles')
      .select('email')
      .or(`username.eq.${email},email.eq.${email}`)
      .limit(1)
      .maybeSingle();
    console.log("profileData:" + JSON.stringify(profileData, null, 2))
    if (userError) throw userError

    const { data: { session }, error } = await supabaseClient.auth.signInWithPassword({
      email: profileData.email,
      password: password,
    })
    if (error) throw error
    console.log("data:" + JSON.stringify(session, null, 2))
    return new Response(JSON.stringify(session), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    })
  }
})
```

This Deno script handles sign-in requests and allows users to choose between usernames and email for login. It leverages Supabase Edge Functions, enhancing the flexibility of your authentication system.

To learn more about testing Supabase Edge Functions, check our dedicated blog post on [Testing Supabase Edge Functions with Deno Test](https://blog.mansueli.com/testing-supabase-edge-functions-with-deno-test).

Here's an example structure for the `public.profiles` table that you can use in your project:

```sql
CREATE TABLE public.profiles (
  id uuid NOT NULL,
  updated_at timestamp with time zone NULL,
  username text NULL,
  full_name text NULL,
  avatar_url text NULL,
  website text NULL,
  email text NULL,
  CONSTRAINT profiles_pkey PRIMARY KEY (id),
  CONSTRAINT profiles_username_key UNIQUE (username),
  CONSTRAINT profiles_id_fkey FOREIGN KEY (id) REFERENCES auth.users (id) ON DELETE CASCADE,
  CONSTRAINT username_length CHECK (char_length(username) >= 3)
);
```

You can read more about creating such a table in [Managing User Data](https://supabase.com/docs/guides/auth/managing-user-data#creating-user-tables).

## Conclusion

In this extensive exploration of user authentication with Supabase, we've equipped you with the knowledge and tools to build secure and flexible login systems for your web applications. By enabling users to choose between username and password or email and password login methods, you've enhanced the user experience.

As you continue to develop and fine-tune your authentication systems, consider the following articles for further insights and optimizations:

* [Comprehensive Guide: Deploying and Debugging Custom Webhooks on Supabase & PostgreSQL](https://blog.mansueli.com/automating-webhooks-supabase-postgresql-guide): Learn how to automate webhooks for seamless integration into your applications.
    
* [Creating Customized i18n-Ready Authentication Emails using Supabase Edge Functions, PostgreSQL, and Resend](https://blog.mansueli.com/creating-customized-i18n-ready-authentication-emails-using-supabase-edge-functions-postgresql-and-resend): Dive into the world of internationalization and create personalized authentication emails.
    
* [How to Boost Supabase Reliability: A Guide to Using Postgres Foreign Data Wrappers](https://blog.mansueli.com/how-to-boost-supabase-reliability-a-guide-to-using-postgres-foreign-data-wrappers): Enhance the reliability of your Supabase applications with foreign data wrappers.
    

These articles will complement your journey to master Supabase authentication and take your web development skills to the next level. Experimentation and customization are key to tailoring authentication systems to your specific project needs.

## References

For additional guidance and resources, consider exploring these valuable references:

* [Supabase Official Documentation](https://supabase.com/docs): Explore Supabase's official documentation for comprehensive insights into this robust backend platform.
    
* [PostgreSQL Official Documentation](https://www.postgresql.org/docs/): Dive into the official documentation of PostgreSQL, the powerful database system at the core of Supabase.
    
* [Next.js Documentation](https://nextjs.org/docs): Access Next.js documentation for further information on this popular React framework.
    
* [Deno Documentation](https://deno.land/manual): Explore Deno documentation for insights into this secure runtime for JavaScript and TypeScript.
    

With these references and your newfound knowledge, you're well-prepared to create exceptional web applications powered by Supabase. Continue your journey in web development, and don't hesitate to explore additional features and customization options offered by Supabase and PostgreSQL.