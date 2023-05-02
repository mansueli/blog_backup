---
title: "User self-deletion with Supabase"
datePublished: Wed Apr 05 2023 16:22:50 GMT+0000 (Coordinated Universal Time)
cuid: clg3we5xe000509mm3sixce9r
slug: user-self-deletion-with-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683040187205/71d8fd9d-e798-410b-9815-e0ff28bf9556.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1683040215743/8c7319ab-3476-484d-b76b-85c2283c24ea.png
tags: javascript, nextjs, supabase, edge-functions

---

Today, we will look at an exciting new project that leverages the power of Supabase and edge functions to build a user management system that includes self-deletion or user invalidation (soft-delete).

If you're unfamiliar with Supabase, it's a powerful open-source platform that makes it easy to build and scale database-backed applications. With Supabase, you can quickly create APIs, handle authentication and authorization, and even deploy serverless functions that run right at the edge of your network.

Our starting point for this project is the user management example that Supabase provides in their [GitHub repository](https://github.com/supabase/supabase/tree/master/examples/user-management/nextjs-ts-user-management). Specifically, we'll use the Next.js user management example as our base code. This project already includes a lot of the functionality you'll need to build a robust user management system, including sign-up, login, and password reset flows.

However, we will take things a step further by adding self-deletion and user invalidation capabilities to the system. This will allow users to delete their accounts if they choose to and also give administrators the ability to disable or invalidate user accounts if necessary.

To achieve this, we'll leverage Supabase's edge functions, which allow you to run serverless code right at the edge of your network, close to your users. This makes it possible to build highly responsive and scalable applications that can handle many user interactions.

So if you're ready to take your user management system to the next level, read on to learn how to expand the Supabase user management example with self-deletion and user invalidation capabilities using edge functions!

### Setting the `profiles` table cascade deletions:

Go into the Supabase Dashboard and edit the profiles table:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680705156682/02af33a8-2fcd-4221-9453-8acb3b6507cb.png align="center")

Then, select the square for the foreign key:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680705945146/cd437633-04fb-41ce-ba55-87958f68082f.png align="center")

Make sure this model looks exactly like this on the delete cascade. What this does is it will delete the row if the source/ reference row is deleted.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680706002504/f1cf418b-da6a-46c3-900b-8bf266bcada0.png align="center")

## Extending the functionality in the web app

We'll need to import the router and set the modal states for the Accounts.tsx file:

```javascript
import { useRouter } from 'next/router'

const [isModalOpen, setIsModalOpen] = useState(false)
const router = useRouter()
```

Then, you can create a delete function that will be responsible to fulfilling the deletion request:

```javascript
  async function deleteAccount() {
    try {
      setLoading(true)
      if (!user) throw new Error('No user')
      await supabase.functions.invoke('user-self-deletion')
      alert('Account deleted successfully!')
    } catch (error) {
      alert('Error deleting the account!')
      console.log(error)
    } finally {
      setLoading(false)
      setIsModalOpen(false)
      // Note that you also will force a logout after completing it
      await supabase.auth.signOut()
      router.push('/')
    }
  }
```

Then, updating the [logout](https://supabase.com/docs/guides/auth/auth-helpers/nextjs#migrating-to-v04x-and-supabase-js-v2) logic:

```javascript
        <button className="button block" onClick={async () => {
                                                  await supabase
                                                  .auth.signOut()
                                                  router.push('/')
                                                  }}>
```

Then, adding the self-deletion feature:

```javascript
  
      <div>
        <button className="button error block" onClick={() => setIsModalOpen(true)}>
          Delete Account
        </button>
      </div>
  
      {isModalOpen && (
        <div className="modal-container">
          <div className="modal-content">
            <h2>Confirm Account Deletion</h2>
            <p>Are you sure you want to delete your account?</p>
            <div>
              <button className="button error" onClick={deleteAccount}>
                Confirm
              </button>
              <button className="button" onClick={() => setIsModalOpen(false)}>
                Cancel
              </button>
            </div>
          </div>
        </div>
      )}
```

You can find the full code for this [Account.tsx](https://github.com/mansueli/subapase-user-self-deletion-nextjs/blob/main/components/Account.tsx) component on GitHub. Now, we'll add a very small [CSS adjustment](https://github.com/mansueli/subapase-user-self-deletion-nextjs/blob/main/styles/globals.css#L34-L36) to make the deletion buttons red:

```css
.button.error {
  background-color: maroon;
}
```

Example:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680710772789/333b81c6-ec41-4b05-b09e-557f0d03c9d0.png align="center")

## User deletion

Now, we will create the edge functions that will perform the user deletion/data invalidation. Both functions should be easily replaceable in the web code.

### Edge Function for deletion:

```javascript
import { serve } from 'https://deno.land/std@0.182.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.14.0'
import { corsHeaders } from '../_shared/cors.ts'

console.log(`Function "user-self-deletion" up and running!`)

serve(async (req: Request) => {
  // This is needed if you're planning to invoke your function from a browser.
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  try {
    // Create a Supabase client with the Auth context of the logged in user.
    const supabaseClient = createClient(
      // Supabase API URL - env var exported by default.
      Deno.env.get('SUPABASE_URL') ?? '',
      // Supabase API ANON KEY - env var exported by default.
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      // Create client with Auth context of the user that called the function.
      // This way your row-level-security (RLS) policies are applied.
      { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
    )
    // Now we can get the session or user object
    const {
      data: { user },
    } = await supabaseClient.auth.getUser()
    // And we can run queries in the context of our authenticated user
    const { data: profiles, error: userError } = await supabaseClient.from('profiles').select('id, avatar_url')
    if (userError) throw userError
    const user_id = profiles[0].id
    const user_avatar = profiles[0].avatar_url
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    )
    const { data: avatar_deletion, error: avatar_error } = await supabaseAdmin
      .storage
      .from('avatars')
      .remove([user_avatar.name])
    if (avatar_error) throw avatar_error
    console.log("Avatar deleted: " + JSON.stringify(avatar_deletion, null, 2))
    const { data: deletion_data, error: deletion_error } = await supabaseAdmin.auth.admin.deleteUser(user_id)
    if (deletion_error) throw deletion_error
    console.log("User & files deleted user_id: " + user_id)
    return new Response("User deleted: " + JSON.stringify(deletion_data, null, 2), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    })
  }
})
```

You can also delete all objects that belong to the user by getting the whole list like this:

```javascript
//Note: this will pick 100 object items if no limit is set.
const { data: list_of_files, error: storageError } = await supabaseClient.storage.from('avatars').list()

if (storageError) throw storageError
const file_urls = []
for (let i = 0; i < list_of_files.length; i++) {
  file_urls.push(list_of_files[i].name)
}

const { data: avatar_deletion, error: avatar_error } = await supabaseAdmin
      .storage
      .from('avatars')
      .remove(file_urls)
```

## User invalidation

### Edge Function to invalidate the user:

```javascript
import { serve } from 'https://deno.land/std@0.182.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.14.0'
import { corsHeaders } from '../_shared/cors.ts'

console.log(`Function "user-invalidation" up and running!`)

serve(async (req: Request) => {
  // This is needed if you're planning to invoke your function from a browser.
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  try {
    // Create a Supabase client with the Auth context of the logged in user.
    const supabaseClient = createClient(
      // Supabase API URL - env var exported by default.
      Deno.env.get('SUPABASE_URL') ?? '',
      // Supabase API ANON KEY - env var exported by default.
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      // Create client with Auth context of the user that called the function.
      // This way your row-level-security (RLS) policies are applied.
      { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
    )
    // Now we can get the session or user object
    const {
      data: { user },
    } = await supabaseClient.auth.getUser()
    // And we can run queries in the context of our authenticated user
    const { data: profiles, error: user_error } = await supabaseClient.from('profiles').select('id')
    if (user_error) throw user_error
    const user_id = profiles[0].id
    // Create the admin client to delete files & user with the Admin API.
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    )
    const { data: invalidate_user, invalidation_error } = await supabaseAdmin.auth.admin.updateUserById(
      '6aa5d0d4-2a9f-4483-b6c8-0cf4c6c98ac4',
      {   
        email: user_id.concat('@deleted-users.example.com'), 
        phone: "", 
        user_metadata: { deleted: true },
        app_metadata:  { deleted: true }
      }
    )
    if (invalidation_error) throw invalidation_error
    const { data: invalidate_profile, invalid_profile_error } = await supabaseAdmin.from('profiles')
    .update({ full_name: '', username: '', avatar_url: '', website:''})
    .eq('id', user_id)
    if (invalid_profile_error) throw invalid_profile_error
    console.log('profile_invalidated:'+JSON.stringify(invalidate_profile, null, 2))
    return new Response("User invalidated: " + JSON.stringify(invalidate_user, null, 2), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    })
  }
})
```

You'll need to deploy either of these functions with the [Supabase CLI](https://supabase.com/docs/reference/cli/supabase-functions-deploy).

```bash
supabase functions deploy user-self-deletion
```

In conclusion, Supabase is a powerful platform to help you build robust and scalable database-backed applications. By leveraging edge functions and the user management example provided by Supabase, we've shown how you can expand your user management system to include self-deletion and user invalidation capabilities.

With these features, you can empower your users to manage their own accounts while giving administrators the tools they need to maintain control and security. I hope this blog post has inspired you to explore the possibilities of Supabase and edge functions in your own projects.