---
title: "Secure Password Verification and Update with Supabase and PostgreSQL"
seoTitle: "Secure Password Verification and Update: Supabase Tutorial"
seoDescription: "Learn how to implement secure password verification and update using Supabase and PostgreSQL. Enhance your app's security with this tutorial."
datePublished: Tue Jul 04 2023 14:00:39 GMT+0000 (Coordinated Universal Time)
cuid: cljocxz9p04c4u8nv4m9dhpwf
slug: secure-password-verification-and-update-with-supabase-and-postgresql
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688478346591/d49cb53e-2bec-4b83-8b92-6f0f38a68eb1.png
tags: postgresql, reactjs, supabase, password-security

---

In the world of web and mobile app development, having a robust backend infrastructure is crucial for ensuring smooth and secure operations. Built on top of PostgreSQL, [Supabase](https://supabase.com/) offers a complete backend platform with real-time capabilities, authentication, and database management. In this blog post, we'll explore how Supabase leverages the power and scalability of PostgreSQL to implement secure password verification and update functionality in your applications.

### Background on User Password Verification

Password verification is a critical aspect of application security. It ensures that only authorized users can access and modify their accounts, protecting sensitive data and maintaining the trust of your users. When implementing password updates, it's essential to verify the old password before allowing any changes. This step prevents unauthorized modifications and enhances the overall security of your application. You can enforce secure password change by going into the [dashboard -&gt; Authentication -&gt; Providers](https://app.supabase.com/project/_/auth/providers):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688146692457/1d1bd5da-de0a-4d08-af22-0496065b8b1e.png align="center")

## Overview of the Implementation

To implement password verification and update functionality using Supabase and PostgreSQL, we'll walk through the code that accomplishes this task. The code consists of three parts: a PostgreSQL function, an Edge function written in TypeScript, and React code that calls the Edge function.

Creating the `verify_user_password` Function in PostgreSQL

The first part involves creating a PostgreSQL function called `verify_user_password`. This function checks if the provided password matches the encrypted password stored in the `auth.users` table. Here's the SQL code for creating the function:

```sql
CREATE OR REPLACE FUNCTION verify_user_password(password text)
RETURNS BOOLEAN SECURITY DEFINER AS
$$
BEGIN
  RETURN EXISTS (
    SELECT id 
    FROM auth.users 
    WHERE id = auth.uid() AND encrypted_password = crypt(password::text, auth.users.encrypted_password)
  );
END;
$$ LANGUAGE plpgsql;
-- You can also protect this function with:
REVOKE EXECUTE ON FUNCTION verify_user_password from anon, authenticated;
```

This function takes a password as input and returns a boolean value indicating whether the password is valid for the current user.

### **Implementing the Edge Function**

In the previous section, we discussed the importance of password verification and update functionality for application security. Now, let's dive into the implementation details using Supabase and PostgreSQL.

To implement the password verification and update functionality, we need to create an Edge function written in TypeScript. This Edge function acts as an intermediary between the frontend and the backend, handling the logic and communication with the Supabase backend.

Here's the code for the Edge function:

```javascript
import { serve } from "https://deno.land/std@0.192.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type",
};

serve(async (req) => {
  // Create a Supabase client with the necessary credentials
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }
  // Create a Supabase client with the necessary credentials
  const supabaseClient = createClient(
    Deno.env.get("SUPABASE_URL") ?? "",
    Deno.env.get("SUPABASE_ANON_KEY") ?? "",
    {
      global: { headers: { Authorization: req.headers.get("Authorization")! } },
      auth: {
        autoRefreshToken: false,
        persistSession: false,
        detectSessionInUrl: false
      }
    }
  );
  console.log("Supabase client created");
  // Fetch the logged-in user from Supabase
  const { data: { user }, error: userError } = await supabaseClient.auth
    .getUser();
  console.log("User fetched", user);
  if (userError) {
    console.error("User error", userError);
    return new Response(JSON.stringify({ error: userError.message }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
      status: 400,
    });
  }
  // Extract the old and new passwords from the request
  const { oldPassword, newPassword } = await req.json();
  console.log("Received old and new passwords", oldPassword, newPassword);
  // Verify the old password using the `verify_user_password` function
  const { data: isValidOldPassword, error: passwordError } =
    await supabaseClient.rpc("verify_user_password", { password: oldPassword });
  console.log("Old password verified", isValidOldPassword);
  if (passwordError || !isValidOldPassword) {
    console.error("Invalid old password", passwordError);
    return new Response(JSON.stringify({ error: "Invalid old password" }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
      status: 400,
    });
  }
  try {
    // Fetch the user's profile data
    const { data: profiles, error: profileError } = await supabaseClient.from(
      "profiles",
    ).select("id, avatar_url");
    console.log("Profile data fetched", profiles);
    if (profileError) throw profileError;
    const user_id = profiles[0].id;
    console.log("User id", user_id);
    // Update the user's password using the Supabase Admin API
    const supabaseAdmin = createClient(
      Deno.env.get("SUPABASE_URL") ?? "",
      Deno.env.get("SUPABASE_SERVICE_ROLE_KEY") ?? "",
      {
        auth: {
          autoRefreshToken: false,
          persistSession: false,
          detectSessionInUrl: false
        }
      }
    );
    console.log("Admin client created");
    // Return a success response to the client
    const { error: updateError } = await supabaseAdmin
      .auth.admin.updateUserById(
        user_id,
        { password: newPassword },
      );
    console.log("Password updated");
    if (updateError) {
      console.error("Update error", updateError);
      return new Response(JSON.stringify({ error: updateError.message }), {
        status: 400,
      });
    }
  } catch (error) {
    console.error("Caught error", error);
    return new Response(JSON.stringify({ error: error }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
      status: 400,
    });
  }
  console.log("Password update successful");
  // Return a success response to the client
  return new Response(
    JSON.stringify({ message: "Password updated successfully" }),
    {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
      status: 200,
    },
  );
});
```

In this code snippet, we start by importing the necessary modules and libraries. Then, we define the CORS headers to allow cross-origin requests. Next, we set up the server to handle incoming requests and extract the authorization header from the request.

We create a Supabase client using the provided credentials and then fetch the logged-in user from Supabase. If there are any errors during the user fetching process, we handle them accordingly.

After extracting the old and new passwords from the request, we verify the old password using the `verify_user_password` function by invoking the `rpc` method on the Supabase client. If the old password is invalid or there are errors during the verification process, we handle them and return an appropriate error response.

If the old password is valid, we proceed to fetch the user's profile data. Once we have the user ID, we create a new Supabase client with admin credentials. Using the Supabase Admin API, we update the user's password to the new password provided.

Finally, we handle any errors that may occur during the password update process and return a successful response if the password update is successful.

This Edge function serves as a crucial component in implementing secure password verification and update functionality using Supabase and PostgreSQL.

### **Calling the Edge Function in React**

To update the password securely, we need to call the Edge function from our React application. Let's dive into the process step by step.

First, we define an asynchronous function called `updatePassword()`. This function handles the password update logic and communicates with the Edge function. Here's an example of how the function looks:

```javascript
javascriptCopy codeasync function updatePassword() {
  try {
    setLoading(true);

    // Ensure there is a user logged in
    if (!user) throw new Error('No user');

    // Validate that the new passwords match
    if (newPassword !== confirmNewPassword) {
      alert('New passwords do not match!');
      return;
    }

    // Call the secure_update_password Edge function
    const { data, error } = await supabase.functions.invoke('secure_update_password', {
      body: {
        "oldPassword": oldPassword,
        "newPassword": newPassword
      }
    });

    if (error) throw error;

    alert('Password updated!');
  } catch (error) {
    alert('Error updating the password!');
    console.log(error);
  } finally {
    setLoading(false);
  }
}
```

In this code, we start by setting the loading state to `true` to indicate that the update process is in progress. We then perform a series of checks:

1. We ensure that a user is logged in. If not, an error is thrown with the message "No user."
    
2. We validate that the new passwords match. If they don't, an alert is displayed, and the function returns without further execution.
    

If both checks pass, we proceed to call the `secure_update_password` Edge function using `supabase.functions.invoke()`. We pass the `oldPassword` and `newPassword` as part of the request body. The function returns a response object that contains `data` and `error` properties.

If an error occurs during the function invocation, we throw the error and display an alert with the message "Error updating the password!" Additionally, the error is logged to the console for further investigation.

Finally, regardless of the outcome, we set the loading state to `false` to indicate that the update process has completed.

Now that we have the `updatePassword()` function defined, we can integrate it into our React component, specifically in the form that allows users to enter their old and new passwords.

```javascript
<div>
  <label htmlFor="old-password">Old Password</label>
  <input
    id="old-password"
    type="password"
    value={oldPassword}
    onChange={(e) => setOldPassword(e.target.value)}
  />
</div>
<div>
  <label htmlFor="new-password">New Password</label>
  <input
    id="new-password"
    type="password"
    value={newPassword}
    onChange={(e) => setNewPassword(e.target.value)}
  />
</div>
<div>
  <label htmlFor="confirm-new-password">Confirm New Password</label>
  <input
    id="confirm-new-password"
    type="password"
    value={confirmNewPassword}
    onChange={(e) => setConfirmNewPassword(e.target.value)}
  />
</div>
```

In this code snippet, we render three input fields: one for the old password, one for the new password, and one to confirm the new password. Each input field is associated with its respective state variable (`oldPassword`, `newPassword`, `confirmNewPassword`). The `onChange` event handlers update the corresponding state variables as the user types in the input fields. You can find the full code including the [User Self-Deletion](https://blog.mansueli.com/user-self-deletion-with-supabase) part in a single [repo](https://github.com/mansueli/subapase-user-self-deletion-nextjs).

### Conclusion

Implementing secure password verification and update functionality is crucial for the overall security of your application. With Supabase and PostgreSQL, you can leverage powerful tools and features to ensure that user passwords are protected and updated securely. By combining the flexibility of Supabase's backend platform with the scalability of PostgreSQL, you can build robust applications that prioritize user security.

In this blog post, we've explored the process of implementing password verification and update functionality using Supabase and PostgreSQL. We've walked through the code snippets that create a PostgreSQL function for password verification, demonstrate its usage in a TypeScript Edge function, and show how it can be called from a React application. Remember to prioritize secure password management in your applications and consider using Supabase and PostgreSQL for your backend needs.

Start building robust and secure applications with Supabase and PostgreSQL today!