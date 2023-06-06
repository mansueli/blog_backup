---
title: "Using Custom Claims for Supabase Storage Policies"
datePublished: Tue Jun 06 2023 14:05:57 GMT+0000 (Coordinated Universal Time)
cuid: clikcsy2p000i09l9hgnxffy2
slug: using-custom-claims-for-supabase-storage-policies
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1685483383346/9ea64330-4015-45d2-bc7d-c1d7e014ca09.png
tags: postgresql, storage, supabase, cloud-storage, row-level-security

---

[Supabase](https://supabase.com/) is an open-source Firebase alternative that allows developers to easily build scalable and secure applications. One of the main features of Supabase is its storage service, which enables users to store and retrieve files. To ensure proper access control and security, Supabase provides storage policies that define the rules for accessing and manipulating files. This blog post will explore how to use custom claims in Supabase storage policies to implement fine-grained access control.

## **Introduction to Supabase Storage Policies**

Supabase storage policies are based on Postgres Row-Level Security (RLS) policies, which determine who can access specific rows of data in a database table. With storage policies, you can extend this concept to files stored in the Supabase storage service. By defining policies, you can enforce rules based on various attributes such as the bucket ID, file path, and user-specific custom claims.

## **Custom Claims for Access Control**

[Custom Claims](https://github.com/supabase-community/supabase-custom-claims) are a powerful feature that provides that allows you to add metadata to a user's session token. These claims can be used to store user-specific information, such as roles, permissions, or any other contextual data relevant to your application.

To leverage custom claims in storage policies, you need to follow these steps:

1. **Define custom claims:**  
    First, define the custom claims you want to use for access control. For example, you can create a custom claim called "team" that stores the teams a user belongs to.
    
2. **Manage custom claims:**  
    You can manage the claims for the user on your backend or [use triggers to manage the relationship](https://blog.mansueli.com/using-triggers-to-map-database-relationships-in-custom-claims) and map them on the custom claims.
    
3. **Use custom claims in storage policies:**  
    Finally, you can use the custom claims in your storage policies to enforce fine-grained access control. The example policy shown below demonstrates how to use a custom claim called "team" to restrict access to files within specific teams.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1685476066582/e1a6d914-016a-4535-a4c4-eb634e7f2943.png align="center")

```pgsql
(
   (bucket_id = 'supa-drive'::text) 
   AND 
   (path_tokens[1] = 'teams'::text) 
   AND 
   -- It is easy to expand this part below for advanced policies
   (path_tokens[2] IN 
   ( SELECT unnest(x.member_of) AS unnest
     FROM json_to_record(
           (get_my_claim('team'::text))::json) x(member_of text[])
          )
   )
)
```

In the above policy, we ensure that the bucket ID is "supa-drive" and the path contains "teams." Furthermore, we check if the user's "team" claim matches any of the teams specified in the path.

In the next part of this blog post, we will explore further examples and considerations for using custom claims in Supabase storage policies.

## **Implementation Example**

Let's consider an example implementation using custom claims for storage policies with Supabase JavaScript SDK. We'll be using [Supabase-JS as a script](https://blog.mansueli.com/using-supabase-js-as-a-script-in-your-terminal), to demonstrate how to upload files using this policy using ArrayBuffer which can be also used with Supabase SDK & [React Native](https://supabase.com/docs/reference/javascript/storage-from-upload).

```javascript
import { createClient } from '@supabase/supabase-js'
import dotenv from 'dotenv' 
import { decode } from 'base64-arraybuffer';
import fs from 'fs';
import path from 'path';

dotenv.config()
// Get Supabase URL from environment variables and log it
const SUPABASE_URL = process.env.SUPABASE_URL
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
const { data: {session} , supa_error} =  await supabase.auth.signInWithPassword({
  email: 'rodrigo@contoso32.com',
  password: '<secret password>',
});
const opt = {
  global: {
    headers: { Authorization: "Bearer "+session.access_token },
  },
};
//This client is logged as rodrigo@contoso32.com
const supa_logged = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, opt);
async function uploadFile(filePath, teamId) {
  // Read the file from the file system
  const file = fs.readFileSync(filePath);
  // Convert the file to a base64 string
  const base64String = file.toString('base64');
  // Extract the file name and extension
  const fileName = path.basename(filePath);
  // Determine the contentType from the local file
  const contentType = file.type;
  // Upload the encoded data to Supabase with the same name and extension
  const { data:upload_data, error: uploadError } = await supa_logged
    .storage
    .from('supa-drive')
    .upload(`teams/${teamId}/${fileName}`, decode(base64String), {
      contentType: contentType,
      upsert: true
    });
  if (uploadError) {
    console.error('Error uploading file. Details:+\n'+JSON.stringify(uploadError,null,2));
  } else {
    console.log('File uploaded successfully. Details:+\n'+JSON.stringify(upload_data,null,2));
  }
}
// Usage example
const filePath = '/Users/mansueli/Desktop/J24K2NVxl.png';
uploadFile(filePath);
```

## Let's take a closer look at the upload function

Now, let's consider the implementation of the `uploadFile` function, which uploads a file to Supabase storage:

```javascript
async function uploadFile(filePath, teamId) {
  // Read the file from the file system
  const file = fs.readFileSync(filePath);
  // Convert the file to a base64 string
  const base64String = file.toString('base64');
  // Extract the file name and extension
  const fileName = path.basename(filePath);
  // Determine the contentType from the local file
  const contentType = file.type;
  // Upload the encoded data to Supabase with the same name and extension
  const { data: upload_data, error: uploadError } = await supa_logged
    .storage
    .from('supa-drive')
    .upload(`teams/${teamId}/+${fileName}`, decode(base64String), {
      contentType: contentType,
      upsert: true,
    });
  if (uploadError) {
    console.error('Error uploading file. Details:\n', JSON.stringify(uploadError, null, 2));
  } else {
    console.log('File uploaded successfully. Details:\n', JSON.stringify(upload_data, null, 2));
  }
}
```

The `uploadFile` function reads the file from the file system, converts it to a base64 string, and extracts the file name and extension. Then, it uses the Supabase client (`supa_logged`) to upload the file to the `supa-drive` bucket which uses custom claims in the policies to authorize the requests. The file is uploaded to the `teams/${teamId}` directory with the same name and extension. The `decode` function converts the base64 string back to the original file data. The `contentType` is determined based on the local file type, and the `upsert` option is set to true to allow updating the file if it already exists.

Finally, the function logs the upload result or any errors that occurred during the upload process.

You could also adapt this function without the use of the array buffer if you are not using it with React Native:

```javascript
const localFile = event.target.files[0]
async function uploadFile(localFile, teamId) {
  const fileName = path.basename(localFile);
  // Determine the contentType from the local file
  const contentType = localFile.type;
  // Uploading the file:
  const { data: upload_data, error: uploadError } = await supa_logged
    .storage
    .from('supa-drive')
    .upload(`teams/${teamId}/+${fileName}`, localFile, {
      contentType: contentType,
      upsert: true,
    });
  if (uploadError) {
    console.error('Error uploading file. Details:\n', JSON.stringify(uploadError, null, 2));
  } else {
    console.log('File uploaded successfully. Details:\n', JSON.stringify(upload_data, null, 2));
  }
}
```

To use the `uploadFile` function, you can provide the file path and the team ID as parameters:

```javascript
const filePath = '/Users/mansueli/Desktop/J24K2NVxl.png';
const teamId = 'example-team-id';
uploadFile(filePath, teamId);
```

This example demonstrates how to use custom claims in Supabase storage policies and leverage them in file upload operations. By incorporating custom claims, you can implement granular access control to ensure that files are securely managed and accessed based on user-specific attributes.

## Conclusion

In conclusion, Supabase storage policies and custom claims provide developers with a powerful mechanism for implementing fine-grained access control and security measures for files stored in the Supabase storage service. By leveraging custom claims, developers can assign contextual information to user sessions and utilize this information in storage policies to enforce access restrictions based on specific attributes or user roles.

With storage policies, developers can define rules that examine attributes such as the bucket ID, file path, or custom claims associated with the user. This level of control allows for precise access management, ensuring that only authorized users or specific teams have access to certain files. By incorporating custom claims into storage policies, developers can enforce complex authorization workflows, manage different levels of permissions, and ensure data privacy.

In summary, by leveraging custom claims in Supabase storage policies, developers can enforce fine-grained access control, ensure data privacy, and create a secure file management system that aligns with their application's requirements. Supabase's comprehensive feature set and developer-friendly SDKs make it an excellent choice for building scalable and secure applications that involve file storage and management. With Supabase, developers can confidently implement access controls and security measures to protect their users' data.

To learn more about Supabase and its storage service, visit [**Supabase.com**](http://Supabase.com). For detailed documentation on storage policies and custom claims, refer to the [**Supabase documentation**](https://supabase.io/docs/guides/storage#policies) for more information.