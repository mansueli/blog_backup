---
title: "Building a Slack Bot for AI-Powered Conversations with Supabase"
seoTitle: "AI-Powered Slack Bot with Supabase: A Complete Guide"
seoDescription: "Learn to create an AI-powered Slack bot using Supabase, Hugging Face, and OpenAI. Boost productivity and user experiences."
datePublished: Tue Oct 10 2023 13:25:25 GMT+0000 (Coordinated Universal Time)
cuid: clnkcu5gd00060al9ezt6hvdl
slug: ai-powered-slack-bot-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1696801001502/df0087a5-9ed1-4ce2-8804-e3fb1dfb7df7.png
tags: ai, postgresql, slack, supabase, edge-functions

---

In today's tech-driven world, integrating AI into your communication tools can enhance productivity and user experiences. Slack, a widely used team collaboration platform, allows developers to create custom bots that automate tasks and provide intelligent responses. In this blog post, we'll guide you through the process of building a Slack bot that harnesses the power of AI, specifically Hugging Face and OpenAI, using the Supabase platform.

**Prerequisites**:

Before we dive into the code, you'll need a few things:

1. **Supabase Account**: Ensure you have an account on [Supabase](http://supabase.com/).
    
2. **Slack Workspace**: You'll require access to a Slack workspace where you can create and install the bot.
    
3. **API Keys**: Obtain API keys for Hugging Face and OpenAI.
    

**Creating the Slack Bot Manifest**:

To begin, you need to define the bot's characteristics in a manifest file (`manifest.yaml`). This file specifies the bot's display name, features, OAuth configuration, and settings. Here's an example manifest for your Slack AI bot:

```yaml
display_information:
  name: SlackAI
features:
  bot_user:
    display_name: SlackAI
    always_online: false
oauth_config:
  scopes:
    bot:
      - app_mentions:read
      - chat:write
      - im:write
      - channels:history
      - commands
settings:
  org_deploy_enabled: false
  socket_mode_enabled: false
  token_rotation_enabled: false
```

We'll interact with the Slack bot using custom webhooks. For detailed instructions, refer to our previous post on [Deploying and Debugging Custom Webhooks on Supabase & PostgreSQL](https://blog.mansueli.com/automating-webhooks-supabase-postgresql-guide).

**Setting Secrets on Supabase CLI**:

You'll need to securely store sensitive information, such as API keys, on Supabase using the CLI. Here's how to do it:

```bash
# Set OpenAI API Key
supabase --project-ref nacho_slacker secrets \
set OPEN_AI=sk-FB9ZJJbluyN4CH0luSQkwG0t6ZZG3wSQ6SFyBj3k8JZDRKSt

# Set Hugging Face API Key
supabase --project-ref nacho_slacker secrets \
set HUGGINGFACE_TOKEN=hf_rSji7SZZuZwN8ZW53tN4CH0TKRtFCN6Cai

# Set Bot OAuth Token
supabase --project-ref nacho_slacker secrets \
set SLACK_TOKEN=xoxb-3882900064320-6000640064577-QDr8Nk637kw6nachoR7bDu9
```

### **Edge Functions: The Soul of Your Slack Bot**

In this section, we'll explore two critical Edge Functions, namely `slack_ai_mentions.ts` and `consume_job.ts`. These functions play vital roles in your Slack bot's functionality.

1. `slack_ai_mentions.ts`: This Edge Function sets up an HTTP server using Deno to listen for Slack events. When the bot receives an app mention, it processes the text adding it into a Queue System in Postgres, which will process the requests and send them back to the user.
    

```ts
import { serve } from 'https://deno.land/std@0.197.0/http/server.ts';
import { WebClient } from 'https://deno.land/x/slack_web_api@6.7.2/mod.js';
import { SupabaseClient } from 'https://esm.sh/@supabase/supabase-js@2';

const slack_bot_token = Deno.env.get("SLACK_TOKEN") ?? "";
const bot_client = new WebClient(slack_bot_token);
const supabase_url = Deno.env.get("SUPABASE_URL") ?? "";
const service_role = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY");
const supabase = new SupabaseClient(supabase_url, service_role);

console.log(`Slack URL verification function up and running!`);

serve(async (req) => {
  try {
    const req_body = await req.json();
    console.log(JSON.stringify(req_body, null, 2));
    const { token, challenge, type, event } = req_body;

    if (type == 'url_verification') {
      return new Response(JSON.stringify({ challenge }), {
        headers: { 'Content-Type': 'application/json' },
        status: 200,
      });
    } else if (event.type == 'app_mention') {
      const { user, text, channel, ts } = event;
      const url_path = text.toLowerCase()
            .includes('code') ? '/code' : '/general';
      const { error } = await supabase.from('job_queue').insert({
        http_verb: 'POST',
        payload: { user, text, channel, ts },
        url_path: url_path
      });

      if (error) {
        console.error(error);
        return new Response(JSON.stringify({ error: error.message }), {
          headers: { 'Content-Type': 'application/json' },
          status: 400,
        });
      }
      await post(channel, 
                 ts, 
                 `Taking a look and will get back to you shortly!`);
      return new Response('', { status: 200 });
    }
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { 'Content-Type': 'application/json' },
      status: 400,
    });
  }
});

async function post(channel: string, thread_ts: string, message: string): 
  Promise<void> {
  try {
    const result = await bot_client.chat.postMessage({
      channel: channel,
      thread_ts: thread_ts,
      text: message,
    });
    console.info(result);
  } catch (e) {
    console.error(`Error posting message: ${e}`);
  }
}
```

1. `consume_job.ts`: This Edge Function manages tasks related to AI model interactions. It handles requests from the slack\_ai\_mentions.ts file and communicates with either Hugging Face or OpenAI to generate text responses. The blog post provides detailed instructions on installing and configuring this Edge Function.
    

```ts
import { serve } from "https://deno.land/std@0.197.0/http/server.ts";
import { WebClient } from "https://deno.land/x/slack_web_api@6.7.2/mod.js";

const slack_bot_token = Deno.env.get("SLACK_TOKEN") ?? "";
const bot_client = new WebClient(slack_bot_token);
const hf_token = Deno.env.get("HUGGINGFACE_TOKEN") ?? "";
const openai_api_key = Deno.env.get("OPEN_AI") ?? "";

console.log("Function that will handle the tasks!");

serve(async (req) => {
  const payload = await req.json();
  const url = new URL(req.url);
  const method = req.method;

  // Extract the last part of the path as the command
  const command = url.pathname.split("/").pop();
  try {
    let generated_text = '';
    if (command == "general") {
      generated_text = await getGeneratedTextFromHuggingFace(payload);
    } else {
      generated_text = await getGeneratedTextFromChatGPT(payload);
    }
    await post(payload.channel, 
               payload.ts, 
               `Thanks for asking: ${generated_text}`);
    return new Response('ok',
      { headers: { "Content-Type": "application/json" }, 
        status: 200
      },
    );
  } catch (error) {
    console.error(error);
    return new Response(
      JSON.stringify({ error: error.message }),
      {
        headers: { "Content-Type": "application/json" },
        status: 500,
      },
    );
  }
});

async function post(
  channel: string,
  thread_ts: string,
  message: string,
): Promise<void> {
  try {
    const result = await bot_client.chat.postMessage({
      channel: channel,
      thread_ts: thread_ts,
      text: message,
    });
    console.info(result);
  } catch (e) {
    console.error(`Error posting message: ${e}`);
  }
}

async function getGeneratedTextFromHuggingFace(payload) {
  let huggingface_url = "";
  let body_content = "";
  huggingface_url = "https://api-inference.huggingface.co/models/mistralai/Mistral-7B-Instruct-v0.1";
  body_content = `[INST] ${payload.text} [/INST]`;
  body_content = JSON.stringify({ "inputs": body_content }, null, 2);

  const huggingface_response = await fetch(huggingface_url, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${hf_token}`,
    },
    body: body_content,
  });
  const huggingface_data = await huggingface_response.json();
  if (huggingface_data.error) {
    console.error(huggingface_data.error);
    throw new Error(huggingface_data.error.message);
  }
  const generated_text = huggingface_data[0]
        .generated_text.split("[/INST]");
  return generated_text.length > 1 ? 
         generated_text[1] : generated_text[0];
}

async function getGeneratedTextFromChatGPT(payload) {
  const headers = {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${openai_api_key}`,
  };

  const openai_url = "https://api.openai.com/v1/chat/completions";

  let body_content = {
    "model": "gpt-3.5-turbo", // Replace with the name of the chat model you want to use
    "messages": [
      {
        "role": "system",
        "content": `You are a detail-oriented, helpful, 
                    and eager to please assistant.`
      },
      {
        "role": "user",
        "content": payload.text
      }
    ]
  };
  const body_content_text = JSON.stringify(body_content, null, 2);
  const openai_response = await fetch(openai_url, {
    method: "POST",
    headers: headers,
    body: body_content_text,
  });
  const openai_data = await openai_response.json();
  if (openai_data.error) {
    console.error(openai_data.error);
    throw new Error(openai_data.error.message);
  }
  return openai_data.choices[0].message.content.trim();
}
```

These Edge Functions are essential components of the Slack bot, allowing it to seamlessly integrate with external AI services and provide intelligent responses in real-time.

**Deploying Edge Functions**:

Next, you'll deploy these two Edge Functions using Supabase CLI:

```bash
supabase functions deploy consume_job \
--project-ref nacho_slacker --no-verify-jwt

supabase functions deploy slack_ai_mentions \
--project-ref nacho_slacker --no-verify-jwt
```

## Storing Secrets Securely in Vault

To ensure the security of sensitive information, it's crucial to utilize Vault for secret management. Here's a step-by-step guide to adding `service_role` and `consumer_function` to Vault:

1. Add `service_role` by using the `service_role` key found in the [Supabase dashboard](https://supabase.green/dashboard/project/_/settings/api).
    
2. Incorporate `consumer_function` into Vault by utilizing the URL of the Edge Function `consume_job` from the [Supabase dashboard](https://supabase.green/dashboard/project/_/functions).
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696800578158/5cae0a7f-5fdc-4c49-9aca-6f4727ac3d42.png align="center")
    

## Configuring Slack Apps

For your bot to seamlessly interact with Slack, you'll need to configure Slack Apps:

1. Navigate to the Slack Apps page.
    
2. Under "Event Subscriptions," add the URL of the `slack_ai_mentions` function and click to verify the URL.
    
3. The Edge function will respond, confirming that everything is set up correctly.
    
4. Add `app-mention` in the events the bot will subscribe to.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696800471362/73b86a1d-3b7a-4014-9411-493ef023a947.png align="center")

## Installing DBDEV and Supa\_Queue

To enhance your Supabase project's capabilities, consider installing the `supabase-dbdev` and `supa_queue` extensions. Here's how you can do it:

### Installing `supabase-dbdev`

```sql
-- Install supabase-dbdev
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
            ('apiKey', 'YOUR_DATABASE_DEV_API_KEY_HERE')::http_header
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

### Installing the `supa_queue` Extension

To further empower your Supabase project, consider installing the `supa_queue` extension. This extension offers a robust queue system for efficiently managing tasks and processes. Here's how you can install it:

```sql
-- Install supa_queue
select dbdev.install('mansueli-supa_queue');
create extension "mansueli-supa_queue" version '1.0.3';
```

**Note**: The `supa_queue` extension is a Trusted Language Extension available at [https://database.dev/mansueli/supa\_queue](https://database.dev/mansueli/supa_queue). It was created based on a previous post we published about [building a simple & robust queue system for PostgreSQL](https://blog.mansueli.com/building-a-queue-system-with-supabase-and-postgresql). This extension enables you to implement a powerful queue system in your Supabase project, which can prove invaluable for managing asynchronous tasks and background processing.

## Installing and Interacting with Your Slack Bot

Now that your Slack bot is ready to roll, it's time to integrate it into your Slack workspace and start interacting with it in two different contexts: one for general use with the Hugging Face model and another for code-related inquiries using the ChatGPT model.

### Installing Your Bot to Slack

1. **Add Your Bot to Slack**: Navigate to your Slack workspace and access the "Apps" section. Search for your bot's name (e.g., "SlackAI") and add it to your workspace.
    
2. **Authorize Bot Permissions**: Ensure you authorize the necessary permissions for your bot, including sending messages, interacting with users, and reading messages.
    
3. **Place Your Bot in a Channel**: Choose a Slack channel where you'd like your bot to operate. Invite your bot to join the channel by mentioning it (e.g., `@SlackAI`).
    

### General Use with Hugging Face Model

In this context, you can utilize the Hugging Face model for general conversations and inquiries. Mention your bot in the channel and ask a question or initiate a conversation. For example:

```sql
@SlackAI How's the weather today?
```

Your bot, powered by the Hugging Face model, will respond with intelligent insights or answers.

### Code-Related Inquiries with ChatGPT Model

When you have code-related questions or need assistance with technical matters, you can invoke the ChatGPT model. Mention your bot again in the channel and frame your inquiry:

```sql
@SlackAI Can you help me with this Python code snippet?
```

Your bot will be ready to assist with coding-related queries, making your team's technical discussions more efficient.

## Conclusion

In this comprehensive exploration of creating a powerful Slack bot with AI capabilities using Supabase, you've gained the knowledge and tools to elevate your team's communication and productivity. By seamlessly integrating AI models and efficient queue management into your Slack bot, you've unlocked a world of automation and intelligent responses.

As you continue to refine and expand your bot's functionalities, consider these top-performing articles for further insights and optimizations:

1. [Using Custom Claims & Testing RLS with Supabase](https://blog.mansueli.com/using-custom-claims-testing-rls-with-supabase): Dive deep into custom claims and role-level security with Supabase to enhance your application's data security.
    
2. [Supabase User Self-Deletion: Empower Users with Edge Functions](https://blog.mansueli.com/supabase-user-self-deletion-empower-users-with-edge-functions): Learn how to implement user self-deletion features using Supabase Edge Functions, empowering users with control over their accounts.
    
3. [Creating Customized i18n-Ready Authentication Emails using Supabase Edge Functions, PostgreSQL, and Resend](https://blog.mansueli.com/creating-customized-i18n-ready-authentication-emails-using-supabase-edge-functions-postgresql-and-resend): Dive into internationalization and create personalized authentication emails with Supabase Edge Functions and PostgreSQL.
    
4. [Testing Supabase Edge Functions with Deno Test](https://blog.mansueli.com/testing-supabase-edge-functions-with-deno-test): Explore best practices for testing your Supabase Edge Functions using Deno, ensuring robust functionality.
    

These articles complement your journey to master Supabase and Slack bot development, offering valuable insights and strategies for enhancing your applications. Experimentation and customization are key to tailoring your bot to meet your project's specific needs.

## References

For additional guidance and resources, consider exploring these valuable references:

* [Supabase Official Documentation](https://supabase.com/docs): Access the official documentation for Supabase to explore the platform's capabilities and features.
    
* [PostgreSQL Official Documentation](https://www.postgresql.org/docs/): Dive into the official documentation of PostgreSQL, the powerful database system at the core of Supabase.
    
* [Deno Documentation](https://deno.land/manual): Explore Deno documentation for insights into this secure runtime for JavaScript and TypeScript.
    

Building a Slack bot infused with AI capabilities offers an exciting opportunity to automate tasks and deliver intelligent responses, ultimately streamlining your team's interactions and fostering a more engaging work environment. Enjoy the journey of experimenting with various AI models and expanding your bot's functionalities!