---
title: "Secure Your Supabase Auth with email_guard: Block Disposable Emails & Enforce Gmail Uniqueness"
seoTitle: "Secure Supabase Auth with email_guard TLE: Block Disposable Emails and Prevent Gmail Duplicates"
seoDescription: "Learn how to block disposable emails in Supabase, prevent Gmail duplicate signups, and install email_guard Trusted Language Extension for secure authentication."
datePublished: Thu Oct 30 2025 15:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cm9x2abc3000109l5xyz12def
slug: secure-supabase-auth-with-email-guard-tle
tags: postgresql, security, supabase, authentication, trusted-language-extensions, block disposable emails, supabase auth hook

---

Building a secure authentication system is not just about strong passwords. It starts with checking the quality of user signups in your [Supabase.com](http://supabase.com/) project. Disposable email addresses often lead to spam accounts, abuse, and fake users that fill up your database. Also, Gmail's flexible rules (like dots and `+tags`) let people make many accounts from one email.

Meet **email_guard**, a Trusted Language Extension (TLE) for PostgreSQL. It works well with Supabase Auth hooks to:
- **Block disposable email domains** with a big, auto-updated list.
- **Normalize Gmail addresses** to stop duplicate signups (by removing dots and `+tags`).
- **Offer easy installation** that fits in any schema.

When you use this with Supabase's own security tools, like [leaked password protection](https://supabase.com/docs/guides/auth/passwords), you build strong protection against bad signups and abuse.

## Why You Need email_guard for Supabase Authentication Security

### The Problem: Disposable Emails and Gmail Tricks

**Disposable email services** (like mailinator.com or guerrillamail.com) let anyone make short-term emails. These are good for tests, but often used to:
- Make spam or bot accounts.
- Skip limits on trials or rates.
- Avoid bans by making new accounts fast.

**Gmail addressing quirks** are tricky too. These all go to the same inbox:
- `john.doe@gmail.com`
- `j.o.h.n.d.o.e@gmail.com`
- `johndoe+promo@gmail.com`
- `johndoe+whatever@gmail.com`

Without fixing this, a user can make many accounts by adding dots or `+tags`. This breaks rules for one account per person and can cheat referral programs or trials.

### The Solution: Smart Email Validation at Signup in Supabase

The `email_guard` TLE gives you:
1. **A blocklist for disposable domains** with over 20,000 known ones (updated weekly).
2. **Gmail normalization** that removes dots and `+tags`, and sets the domain to `gmail.com`.
3. **A helper for Supabase Auth hooks** that checks before creating a user.

All this runs in PostgreSQL, so it is fast, safe, and clear. For more on Supabase auth, see the [Supabase Auth docs](https://supabase.com/docs/guides/auth).

---

## Understanding Trusted Language Extensions (TLE) in Supabase

Before installation, let's quickly explain what a Trusted Language Extension is and why it helps.

### What is a TLE?

A **Trusted Language Extension (TLE)** is a PostgreSQL add-on written in a safe language (like PL/pgSQL or PL/Python). You can install it without special admin rights. This is key for hosted setups like Supabase, where you lack full access.

TLEs come from [database.dev](https://database.dev), the package manager for PostgreSQL. This makes them:
- **Easy to install** with the Supabase CLI.
- **Version-controlled** for safe updates.
- **Flexible** to place in any schema.
- **Safe for production**.

Learn more in Supabase's [Trusted Language Extensions for Postgres blog post](https://supabase.com/blog/pg-tle) or the [extensions docs](https://supabase.com/docs/guides/database/extensions).

### Why Use TLEs for Security in Supabase?

With security logic in a TLE:
- **It runs in the database**, near your data.
- **It stays the same** for all apps and calls.
- **You can check it** with version history.
- **It cannot be skipped** by app code.

This is great for auth checks, data rules, and policies.

---

## Step 1: Installing the TLE Infrastructure on Supabase

Before adding `email_guard`, set up the `pg_tle` extension (already on Supabase) and the `dbdev` tool for easy install.

### A. Set Up dbdev and Supabase CLI (If Needed)

If not done yet:
1. **Install dbdev CLI**: Follow the [dbdev getting started guide](https://supabase.github.io/dbdev/getting-started/).
2. **Install Supabase CLI**: Check the [Supabase CLI docs](https://supabase.com/docs/guides/cli).
3. **Link your project**: Run `supabase link` to connect to your Supabase database.

Supabase has `pg_tle` ready, so create it:

```sql
CREATE EXTENSION IF NOT EXISTS pg_tle;
```

### B. Generate the Migration with dbdev

Use dbdev to get the latest `email_guard` (version 0.3.1 or newer) in your migrations:

```bash
dbdev add \
  -o ./supabase/migrations/ \
  -v 0.3.1 \
  -s extensions \
  package \
  -n mansueli@email_guard
```

**What this does:**
- Makes a new migration file in `./supabase/migrations/`.
- Puts the extension in the `extensions` schema (good for Supabase).
- Uses version 0.3.1 (change for new versions).

Adjust the `-o` path if your folder is different.

### C. Apply the Migration

Push to your Supabase database:

```bash
supabase db push
```

Done! The extension is installed with the full blocklist.

For more on managing extensions, see [Supabase database extensions docs](https://supabase.com/docs/guides/database/extensions).

---

## Step 2: Understanding What You Just Installed

The `email_guard` extension adds objects in your schema (like `extensions`):

### Table: disposable_email_domains

```sql
-- Holds over 20,000 disposable email domains
CREATE TABLE extensions.disposable_email_domains (
  domain text PRIMARY KEY,
  CONSTRAINT disposable_email_domains_domain_lowercase
    CHECK (domain = lower(domain))
);
```

It fills with domains like:
- `mailinator.com`
- `guerrillamail.com`
- `10minutemail.com`
- And many more.

### Function: normalize_email(text)

```sql
-- Example: normalize_email('J.o.h.n.Doe+promo@gmail.com')
-- Returns: 'johndoe@gmail.com'
SELECT extensions.normalize_email('J.o.h.n.Doe+promo@gmail.com');
```

**What it does:**
- Makes lowercase.
- For Gmail/Googlemail:
  - Removes dots from the start.
  - Cuts after `+` (and the `+`).
  - Sets domain to `gmail.com`.
- For others: Just lowercase.

### Function: is_disposable_email_domain(text)

```sql
-- Check if a domain is disposable
SELECT extensions.is_disposable_email_domain('mailinator.com'); -- true
SELECT extensions.is_disposable_email_domain('gmail.com');      -- false
```

**Smart check:**
- Looks at parent domains (e.g., `sub.mailinator.com` matches).
- Fast with index.

### Function: is_disposable_email(text)

```sql
-- Easy check for full emails
SELECT extensions.is_disposable_email('user@guerrillamail.com'); -- true
```

### Hook Helper: hook_prevent_disposable_and_enforce_gmail_uniqueness(jsonb)

This is the key part! For Supabase Auth hooks, it:
1. **Checks disposable domains** → Sends 403 error if yes.
2. **Normalizes Gmail** → Checks if the same normalized email exists.
3. **Sends 409 error** if duplicate.
4. **Allows** phone signups or non-email.

---

## Step 3: Wire Up the Supabase Auth Hook for Email Validation

Now, connect it to signups.

### Navigate to Auth Hooks in Dashboard

1. Go to your [Supabase Dashboard](https://supabase.com/dashboard).
2. Pick your project.
3. Go to **Authentication** → **Hooks**.
4. Turn on **Before User Created**.

### Configure the Hook

Pick:
- **Hook Type**: `Postgres Function`.
- **Schema**: `extensions` (or your choice).
- **Function**: `hook_prevent_disposable_and_enforce_gmail_uniqueness`.

![Supabase Auth Hook Configuration for email_guard](https://cdn.hashnode.com/res/hashnode/image/upload/v1730300100000/auth-hook-setup.png)

Done! The hook is on. See [Supabase Auth Hooks docs](https://supabase.com/docs/guides/auth/auth-hooks) for details.

### What Happens During Signup

When signing up, the hook checks first:

```javascript
// Try with disposable email
supabase.auth.signUp({
  email: 'test@mailinator.com',
  password: 'secure_password_123'
})
// ❌ Error: { message: "Disposable email addresses are not allowed", status: 403 }

// Try with duplicate Gmail
// If johndoe@gmail.com exists
supabase.auth.signUp({
  email: 'j.o.h.n.d.o.e+test@gmail.com',
  password: 'secure_password_123'
})
// ❌ Error: { message: "A user with this normalized email already exists", status: 409 }

// Valid email
supabase.auth.signUp({
  email: 'alice@example.com',
  password: 'secure_password_123'
})
// ✅ User created
```

---

## Step 4: Testing Your email_guard Setup on Supabase

Check if it works.

### Test 1: Check Disposable Email Detection

```sql
-- True
SELECT extensions.is_disposable_email('user@mailinator.com');

// False
SELECT extensions.is_disposable_email('user@gmail.com');
```

### Test 2: Test Gmail Normalization

```sql
-- All return 'johndoe@gmail.com'
SELECT extensions.normalize_email('John.Doe@gmail.com');
SELECT extensions.normalize_email('j.o.h.n.d.o.e@googlemail.com');
SELECT extensions.normalize_email('johndoe+promo@gmail.com');
```

### Test 3: Simulate the Hook

```sql
-- Disposable reject
SELECT extensions.hook_prevent_disposable_and_enforce_gmail_uniqueness(
  '{"user": {"email": "test@guerrillamail.com"}}'::jsonb
);
-- Error: "Disposable email addresses are not allowed"

// Gmail duplicate (after adding test user)
SELECT extensions.hook_prevent_disposable_and_enforce_gmail_uniqueness(
  '{"user": {"email": "j.o.h.n.d.o.e+test@gmail.com"}}'::jsonb
);
-- Error: "A user with this normalized email already exists"
```

---

## Step 5: Combining with Supabase's Built-in Protections

`email_guard` is stronger with Supabase features.

### Leaked Password Protection

Supabase checks against [HaveIBeenPwned](https://haveibeenpwned.com/). Turn it on in **Dashboard → Authentication → Password Protection**.

```javascript
// Leaked password
supabase.auth.signUp({
  email: 'alice@example.com',
  password: 'password123'  // Leaked
})
// ❌ Error: { message: "Password has been leaked", status: 422 }
```

---

## Keeping the Blocklist Current in email_guard

`email_guard` updates itself.

### How Updates Work

A GitHub workflow:
1. **Runs every week** (Mondays).
2. **Gets new list** from [disposable-email-domains repo](https://github.com/disposable-email-domains/disposable-email-domains).
3. **Makes upgrade script** if changed.
4. **Updates version** (e.g., 0.3.1 to 0.3.2).
5. **Saves changes** auto.

### Upgrading to the Latest Version

For new version:

```bash
# Make upgrade migration
dbdev add \
  -o ./supabase/migrations/ \
  -v 0.3.2 \  # New
  -s extensions \
  package \
  -n mansueli@email_guard

# Apply
supabase db push
```

It adds new domains and keeps data. Watch releases on [GitHub repo](https://github.com/mansueli/email_guard).

---

## Advanced Usage & Customization for Supabase email_guard

### Custom Domain Blocking

Block extra domains:

```sql
-- Add one
INSERT INTO extensions.disposable_email_domains (domain)
VALUES ('suspicious-domain.com')
ON CONFLICT DO NOTHING;

-- Remove one
DELETE FROM extensions.disposable_email_domains
WHERE domain = 'some-domain.com';
```

### Checking Existing Users

Audit users:

```sql
-- Find disposable
SELECT id, email
FROM auth.users
WHERE extensions.is_disposable_email(email);

-- Find Gmail duplicates
WITH normalized AS (
  SELECT 
    id,
    email,
    extensions.normalize_email(email) AS normalized_email
  FROM auth.users
  WHERE email ILIKE '%@gmail.com'
     OR email ILIKE '%@googlemail.com'
)
SELECT 
  normalized_email,
  array_agg(email) AS duplicate_emails,
  count(*) AS duplicate_count
FROM normalized
GROUP BY normalized_email
HAVING count(*) > 1;
```

### Custom Hook Logic

Make your own hook:

```sql
CREATE OR REPLACE FUNCTION public.my_custom_signup_hook(event jsonb)
RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
  user_email text;
BEGIN
  user_email := event->'user'->>'email';

  IF extensions.is_disposable_email(user_email) THEN
    RAISE EXCEPTION 'Nice try! No disposable emails here.'
      USING HINT = 'Please use a permanent email address',
            ERRCODE = 'P0001';
  END IF;

  -- Add custom rules

  RETURN event;
END;
$$;
```

---

## Performance Considerations for email_guard in Supabase

### Benchmarking

Functions are fast:

```sql
-- Domain check: ~0.1ms
SELECT extensions.is_disposable_email_domain('mailinator.com');

-- Normalize: ~0.05ms
SELECT extensions.normalize_email('j.o.h.n.d.o.e+test@gmail.com');

-- Full hook: ~1-2ms
```

### Index Optimization

Extension adds index on `auth.users(email)`. For big databases:

```sql
-- Partial index for Gmail
CREATE INDEX IF NOT EXISTS users_gmail_normalized_idx 
ON auth.users (extensions.normalize_email(email))
WHERE email ILIKE '%@gmail.com' OR email ILIKE '%@googlemail.com';
```

---

## Troubleshooting email_guard on Supabase

### Hook Not Triggering

**Check setup:**
```sql
SELECT * FROM supabase_functions.hooks
WHERE hook_name = 'before_user_created';
```

**Check rights:**
```sql
GRANT EXECUTE ON FUNCTION extensions.hook_prevent_disposable_and_enforce_gmail_uniqueness(jsonb)
TO supabase_auth_admin;
```

### False Positives

If wrong block:

```sql
DELETE FROM extensions.disposable_email_domains
WHERE domain = 'legitimate-domain.com';
```

Report to [blocklist repo](https://github.com/disposable-email-domains/disposable-email-domains).

### Migration Conflicts

Use different schema:

```bash
dbdev add \
  -o ./supabase/migrations/ \
  -v 0.3.1 \
  -s email_guard \
  package \
  -n mansueli@email_guard
```

Update hook to `email_guard.hook_prevent_disposable_and_enforce_gmail_uniqueness`.

---

## Security Best Practices for Supabase Authentication

### Defense in Depth

Add layers:
1. **Verify emails**: Make users confirm.
2. **CAPTCHA**: Use hCaptcha on forms.
3. **Rate limits**: Stop many tries per IP.
4. **Review accounts**: Flag odd patterns.

### Monitoring

Track blocks:

```sql
CREATE TABLE IF NOT EXISTS blocked_signups (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  email text NOT NULL,
  reason text NOT NULL,
  created_at timestamptz DEFAULT now()
);

-- Log in hook (add yourself)
```

### Privacy Considerations

- **Avoid logging full emails**.
- **Hash data** for stats.
- **Follow GDPR/CCPA** for stored info.

---

## Conclusion: Building a Secure Foundation with email_guard on Supabase

Authentication is the door to your app. Secure it with layers. The `email_guard` TLE gives a simple way to block disposable emails and stop Gmail duplicates in Supabase.

With Supabase tools like leaked password checks, email verification, and rate limits, you get a strong system that:
- ✅ **Blocks bad signups** auto.
- ✅ **Stops abuse** without work.
- ✅ **Grows with your app**.
- ✅ **Updates weekly**.

It runs in the database, so it is clear, checked, and hard to skip.

### Next Steps

1. **Install the extension** as shown.
2. **Turn on the auth hook** in dashboard.
3. **Test** with disposable and Gmail tests.
4. **Watch logs** for blocks.
5. **Update** when new versions come.

For more, see:
- [email_guard GitHub repository](https://github.com/mansueli/email_guard)
- [Supabase Auth Hooks documentation](https://supabase.com/docs/guides/auth/auth-hooks)
- [Trusted Language Extensions blog post](https://supabase.com/blog/pg-tle)
- [database.dev package registry](https://database.dev)

Check my previous posts: [Building User Authentication with Username and Password Using Supabase](https://blog.mansueli.com/building-user-authentication-with-username-and-password-using-supabase) and [Streamlining PostgreSQL Function Management with Supabase](https://blog.mansueli.com/streamlining-postgresql-function-management-with-supabase).
