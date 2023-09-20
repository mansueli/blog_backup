---
title: "Allowing users to invite others with Supabase Edge Functions"
datePublished: Wed May 10 2023 14:30:41 GMT+0000 (Coordinated Universal Time)
cuid: clhhssrn900j975nv4nlw395l
slug: allowing-users-to-invite-others-with-supabase-edge-functions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683651972132/ad08b439-1681-44e7-8e0e-ff8d60d1ed8a.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1683666309953/8d0a5301-5d33-486f-bd1c-07035bebc83f.png
tags: postgresql, javascript, supabase, edge-functions, custom-claims

---

In this blog post, we will discuss how to allow users to invite other users to your application using [Supabase](https://supabase.com/) edge functions. We will focus on using custom claims and Supabase Edge Function to achieve this functionality.

## Prerequisites

Please ensure you have already set up custom claims in your Supabase project by following the instructions in the [Supabase-custom-claims repository](https://github.com/supabase-community/supabase-custom-claims). You can read more about [custom claims and testing RLS](https://blog.mansueli.com/using-custom-claims-testing-rls-with-supabase) in our previous post.

## Creating claims:

We are going to assume a simple Teams table:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1683653909815/235ea2ab-7c48-4233-840b-f84af514dcbc.png align="center")

Where there are multiple relationships between users and teams:

* A user can belong to more than one team
    
* A user can be the owner of more than one team
    
* Only team owners can invite users to their teams
    

### Setting up the claims for the admin user:

The admin will have both the claims of being a member of the team and the claim of being a team owner.

```pgsql
-- Our admin is owner fo two teams 1 & 2
select set_claim('897064a9-8a57-4b3d-8468-ca0d35c72d44', 
                 'team_owner', 
                 '[1,2]'::jsonb);

-- I don't recommend the approach below for most cases.
-- We are using here to expand the possibilities of how to use more advanced claims. 
select set_claim('897064a9-8a57-4b3d-8468-ca0d35c72d44', 
       'team', 
       '{"member_of":[1,2,3]}'::jsonb);
```

Note that it is easier to set up just an array for checking memberships.

### Setting Claims for a member of the team:

To set claims for a member of the team, use the following SQL query:

```pgsql
select set_claim('8f1eb286-01dd-4412-ab90-5a4d20be04f1', 
        'team', 
        '{"member_of":[3]}'::jsonb);
-- You can see that it would be easier to set up just an array for checking memberships e.g
select set_claim('8f1eb286-01dd-4412-ab90-5a4d20be04f1', 
        'team', 
        '[3]'::jsonb)
```

### ⚠️ Do NOT allow users to set their own claims ⚠️

The `set_claims` the function should only be allowed from a server/admin standpoint. It is important to protect it with the following:

```pgsql
REVOKE EXECUTE ON FUNCTION set_claims FROM anon, authenticated;
```

### Helper Admin functions

Here is a helper function that retrieves a user ID by their email:

```pgsql
CREATE OR REPLACE FUNCTION get_user_id_by_email(email TEXT)
RETURNS TABLE (id uuid)
SECURITY definer
AS $$
BEGIN
  RETURN QUERY SELECT au.id FROM auth.users au WHERE au.email = $1;
END;
$$ LANGUAGE plpgsql;
-- To protect this function, use the following SQL query:
REVOKE EXECUTE ON FUNCTION get_user_id_by_email FROM anon, authenticated;
```

Updating the `is_admin` from custom claims to accept the service\_role as admin:

```pgsql
CREATE OR REPLACE FUNCTION is_claims_admin() RETURNS "bool"
  LANGUAGE "plpgsql" 
  AS $$
  BEGIN
    IF session_user = 'authenticator' THEN
      IF extract(epoch from now()) > coalesce((current_setting('request.jwt.claims', true)::jsonb)->>'exp', '0')::numeric THEN
        return false; -- jwt expired
      END IF;
      
      IF (current_setting('request.jwt.claims', true)::jsonb)->>'role' = 'service_role' THEN
        return true; -- user has the 'service_role'
      END IF;
      IF coalesce((current_setting('request.jwt.claims', true)::jsonb)->'app_metadata'->'claims_admin', 'false')::bool THEN
        return true; -- user has claims_admin set to true
      ELSE
        return false; -- user does NOT have claims_admin set to true
      END IF;
      --------------------------------------------
      -- End of block 
      --------------------------------------------
    ELSE -- not a user session, probably being called from a trigger or something
      return true;
    END IF;
  END;
$$;
```

I believe that if the function is being called using the service\_role key, then it is safe to assume that the user should be allowed as an admin. This is why we updated this function above.

## The Invite Edge function

To allow users to invite other users to your Supabase application, you can create a custom Supabase [Edge Function](https://supabase.com/docs/guides/functions) called "invite." This function should be called when a user wants to invite another user to their team.

The function then extracts the email and team ID from the request data, creates a Supabase client with the Auth context of the logged-in user, and gets the user's session or user object. It then runs a query to check if the user is an owner of the specified team. If the user is not an owner of the team, the function returns a 403 Forbidden response.

The function then creates a Supabase Admin client, which has elevated privileges. It uses this client to invite the specified email to the project and retrieve the new user's ID. It then sets custom claims for the user, indicating that they are a member of the specified team. We'll be using this [cors.ts](https://github.com/mansueli/supabase-user-self-deletion-nextjs/blob/main/supabase/functions/_shared/cors.ts) file for handling the CORS headers:

```sql
export const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}
```

Here's the code for the "invite" function:

```javascript
import { serve } from 'https://deno.land/std@0.182.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.21.0'
import { corsHeaders } from '../_shared/cors.ts'

console.log(`Function "invite" up and running!`)

serve(async (req: Request) => {
  // This is needed if you're planning to invoke your function from a browser.
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  try {
    const request_data = await req.json();
    const {email, team_id} = request_data;
    // Create a Supabase client with the Auth context of the logged in user.
    const supabaseClient = createClient(
      // Supabase API URL - env var exported by default.
      Deno.env.get('SUPABASE_URL') ?? '',
      // Supabase API ANON KEY - env var exported by default.
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      // Create client with Auth context of the user that called the function.
      // This way your row-level-security (RLS) policies are applied.
      { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
    );
    // Now we can get the session or user object
    const {
      data: { user },
    } = await supabaseClient.auth.getUser();
    if (!user) {
      return new Response("User not logged in", {
        headers: corsHeaders,
        status: 403,
      });}
    console.log(`User data: ${JSON.stringify(user)}`);
    // And we can run queries in the context of our authenticated user
    const { data: claim_data, error: userError } = await supabaseClient.rpc('get_my_claim', { claim: 'team_owner'});
    if (userError) {
      console.log("uerror:"+userError);
      throw userError;
    }
    const claims = Object.values(claim_data);
    if (!claims.includes(team_id)){
      return new Response("User is not an owner of this team", {
        headers: corsHeaders,
        status: 403,
      })
    }
    //Setting up the Admin client
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    )
    //Invinting a user to the project
    const { data:invitation_data, error:invitation_data_error } = await supabaseAdmin.auth.admin.inviteUserByEmail(email);
    if (invitation_data_error){
      console.log('invitation data error:'+JSON.stringify(invitation_data_error));
      throw invitation_data_error;
    }
    const {data:user_id, error: user_id_error} = await supabaseAdmin.rpc('get_user_id_by_email', {email: email});
    if (user_id_error) {
      console.log('Getting user error:' + JSON.stringify(user_id_error));
      throw user_id_error;
    }
    const new_user_id = user_id[0]['id'];

// Setting custom claims for the user
    // You can also auto-confirm these users (not mantatory)
    const { data: user_confirmation, error: user_confirmation_error } = await supabaseAdmin.auth.admin.updateUserById(
      new_user_id,
      { email_confirm: true }
    );
     if (user_confirmation_error) {
       console.log('updateUserById error:' + JSON.stringify(user_confirmation_error));
       throw user_confirmation_error;
    }
    //Now, we can set the custom claims for the user:
    console.log('new_user_id:'+new_user_id);
    const { data:claim_set_data, error: claim_set_error } = await supabaseAdmin
        .rpc('set_claim', 
            {'uid':new_user_id, 
             'claim': 'team',
             'value': `{\"member_of\":[${team_id}]}`
            });
    if(claim_set_error) {
      console.log('claim_set' + JSON.stringify(claim_set_error));
      throw claim_set_error;
    }

    // You can also use the admin client to directly create the user's account with password:
    /*
    const { data:invitation_data, error } = await supabase.auth.admin.createUser({
      email: 'user@email.com',
      password: 'password',
      user_metadata: { name: 'Yoda' }
    })
    */

    const response = `{'data': user ${email} invited to the project ${team_id}}`;
    return new Response(JSON.stringify(response, null, 2), {
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

The function returns a successful response indicating that the user has been invited to the team.

### Conclusion

In conclusion, this blog post has demonstrated how to implement a user-invite functionality in a Supabase application using custom claims and Supabase Edge Functions. We have covered setting up custom claims for both admin users and team members, creating helper admin functions, and updating the `is_admin` function to accept the `service_role` as admin.

The core of the implementation lies in the `invite` Edge Function, which checks whether the requester is a team owner before inviting a new user to the specified team. The function uses a Supabase client with the Auth context of the logged-in user, ensuring that row-level security policies are applied. Additionally, an Admin client is created to handle elevated privileges, such as inviting users and setting custom claims.

By following the steps outlined in this post, developers can enable a secure and efficient way of allowing users to invite others to their teams in a Supabase application.