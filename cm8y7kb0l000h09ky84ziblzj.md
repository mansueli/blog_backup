---
title: "SupaBrain – When Supabase Got Too Fast"
seoTitle: "SupaBrain: Supabase's Speed Revolution"
seoDescription: "Discover SupaBrain: a playful PostgreSQL extension merging speed with BrainFuck language for the ultimate quirky coding challenge"
datePublished: Tue Apr 01 2025 08:00:23 GMT+0000 (Coordinated Universal Time)
cuid: cm8y7kb0l000h09ky84ziblzj
slug: supabrain-when-supabase-got-too-fast
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1742988413136/b063d385-a2e7-4395-ae8c-1830c2f85d46.png
tags: postgresql, brainfuck, supabase, plv8

---

In the world of databases, speed is usually a virtue. But what happens when your database is so lightning fast that it leaves developers questioning reality? Enter [Supabase](https://supabase.com/), a platform so performant that queries finish their work before you even have time to sip your coffee. To playfully address this `problem`, we built **SupaBrain**—a Trusted Language Extension (TLE) for PostgreSQL that brings the mind-bending power of BrainFuck right into your database. Who needs traditional SQL when you can execute a language designed to baffle even the most seasoned engineers?

## Why BrainFuck? Why Not?

PostgreSQL has long been celebrated for its extensibility. So why stick with conventional languages like SQL, `PL/pgSQL`, or even Python when you can support something as esoteric as BrainFuck? With its minimalist syntax (just 8 commands) and unparalleled debugging experience where every error is considered a `feature`, BrainFuck offers:
  
- **Minimalist syntax:** Finally, a language where `SELECT *` is far too verbose.
- **Unparalleled debugging experience:** Errors in BrainFuck keep you on your toes—every mistake is practically an art form.
- **Legacy compatibility:** Perfect for those rare moments when you need to integrate with systems dating back to the era of teleprinters.

## Introducing the `brainfuck()` Function

The crown jewel of SupaBrain is the `brainfuck(code text, input text)` function, which lets you execute BrainFuck code directly within PostgreSQL using PLV8. Here’s a simple example:

```sql
SELECT brainfuck(',[.[-],]', 'Hello!');
```

This snippet will output `"Hello!"` -- right after PostgreSQL takes a brief existential pause.

### The Code Behind the Magic

Below is the full implementation of the `brainfuck()` function:

```sql
CREATE OR REPLACE FUNCTION brainfuck(code text, input text)
 RETURNS text
 LANGUAGE plv8
 IMMUTABLE STRICT
AS $function$
  var tokens = {
    ".": "output += String.fromCharCode(tape[pointer]);",
    ",": "tape[pointer] = input.length ? input.shift().charCodeAt(0) : 0;",
    "<": "pointer = pointer ? pointer - 1 : 255;",
    ">": "pointer = (pointer + 1) % 256;",
    "-": "tape[pointer] = tape[pointer] ? tape[pointer] - 1 : 255;",
    "+": "tape[pointer] = tape[pointer] ? (tape[pointer] + 1) % 256 : 1;",
    "[": "while(tape[pointer]){",
    "]": "}"
  };
  if (!input) {
    input = "";
  }
  var inner_code = "";
  for (var i=0; i<code.length; i++){
    if (tokens[code[i]]) {
      inner_code += tokens[code[i]] + "\n";
    }
  }
  var js_code = "input = Array.from(input + '');\nvar output = '';\nvar tape = [];\nvar pointer = 0;\n" + 
    inner_code + "return output;";
  var fn = new Function("input", js_code);
  return fn(input.split(','));
$function$;
```

This implementation maps each BrainFuck token to a corresponding JavaScript snippet, effectively creating a bridge between BrainFuck and PostgreSQL. By compiling BrainFuck code into JavaScript, SupaBrain enables the execution of esoteric programs in your database.

## Real-World "Applications"

While SupaBrain is clearly a tongue-in-cheek experiment, its potential applications are as imaginative as they are impractical:

- **Query Optimization:**  
  Tired of instantaneous results? Implement a BrainFuck-based execution engine to slow things down. Your queries now run with the deliberation of geological time scales.
  
- **Data Encryption (Sort of):**  
  Secure your data not with advanced cryptography, but with sheer unreadability. If no one can decipher your BrainFuck logic, your secrets are safe by default.
  
- **Hiring Filter:**  
  Ever been frustrated by a candidate's inability to appreciate complexity? If someone voluntarily writes queries in BrainFuck, congratulations—you’ve likely found your next 10x engineer.

## Performance Considerations (or Lack Thereof)

Traditional SQL queries finish in milliseconds. With BrainFuck, you’re invited to experience time in a whole new dimension.

**Why do:**

```sql
-- 0.1ms 
SELECT 1;
```

**When you can:**

```sql
-- 1.7 billion years
SELECT brainfuck('[-]+>[-]+>[-]+>[-]+>[-]+>[-]+<<[>>>[-]>>[-]>[-]<<<<<[>>>>+>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<[-]>>>[-]<<<<<[>>+>>>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[>>>>>[-]>>[-]>[-]<<<<<<<<<<<<<<<[>>>>>>>>>>>>>>+>+<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>-]<<[-]>>>[-]<<<<<<<<<<<<<<<[>>>>>>>>>>>>+>>>+<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[>>>>>[-]+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++>[-]+>[-]+>[-]>>[-]>[-]<<<<<[>>>>+>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<[-]>>>[-]<<<<<[>>+>>>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[<<<<<<<<.>>>>>>>>[-]]>>[[-]]<<<<<<<<<<<<<<<[-]]>>[[-]]>>>>[-]>>[-]>[-]<<<<<<<<<<<<<<<<<<<<<<<<<<[>>>>>>>>>>>>>>>>>>>>>>>>>+>+<<<<<<<<<<<<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>>>>>>>>>>>-]<<[-]>>>[-]<<<<<<<<<<<<<<<<<<<<<<<<<<[>>>>>>>>>>>>>>>>>>>>>>>+>>>+<<<<<<<<<<<<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>>>>>>>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[>>>>>[-]++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++>[-]+>[-]+>[-]>>[-]>[-]<<<<<[>>>>+>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<[-]>>>[-]<<<<<[>>+>>>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[<<<<<<<<.>>>>>>>>[-]]>>[[-]]<<<<<<<<<<<<<<<[-]]>>[[-]]>>>>[-]>>[-]>[-]<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<[>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>+>+<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>-]<<[-]>>>[-]<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<[>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>+>>>+<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[>>>>>[-]++++++++++>[-]+>[-]+>[-]>>[-]>[-]<<<<<[>>>>+>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<[-]>>>[-]<<<<<[>>+>>>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]>>>>[-]>[-]<<<<<<[>>>>>+>+<<<<<<-]>>>>>>[<<<<<<+>>>>>>-]>[-]+>[-]>[-]<<<<[>>>+>+<<<<-]>>>>[<<<<+>>>>-]<[<[-]>[-]]<<<[<<<<<<<<.>>>>>>>>[-]]>>[[-]]<<<<<<<<<<<<<<<[-]]>>[[-]]<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<[-]]>>[[-]]<<<<<<<<[-]+>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>[-]+>[-]>>[-]>[-]<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<[>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>+>+<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>[<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>-]<<[-]>>>[-]<<<<<[>>+>>>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<<[>[<<[-]+>>[-]]<[-]]<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<[-]>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>[-]<<<<<[<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<+>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>+<<<<<-]>>>>>[<<<<<+>>>>>-]<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<]
, '');``` 

The performance trade-off is clear: if you desire to truly savor the passage of time, SupaBrain’s execution will ensure that every moment counts. And don’t worry about scalability—if anything, it scales logarithmically… downwards.

## Installation: Making PostgreSQL Regret Its Flexibility

Before proceeding, ensure you have the following installed:
- **PLV8:** Required for running JavaScript code within PostgreSQL.
- **Database.dev:** Provides access to the PostgreSQL package manager and trusted language extensions (TLE).

### Installing the Trusted Language Extension (TLE)

If you don't have the TLE installed already, follow these steps:

```sql

-- Install TLE if not already installed
create extension if not exists http with schema extensions;
create extension if not exists pg_tle;
drop extension if exists "supabase-dbdev";
select pgtle.uninstall_extension_if_exists('supabase-dbdev');
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
            ('apiKey', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InhtdXB0cHBsZnZpaWZyYndtbXR2Iiwicm9sZSI6ImFub24iLCJpYXQiOjE2ODAxMDczNzIsImV4cCI6MTk5NTY4MzM3Mn0.z2CN0mvO2No8wSi46Gw59DFGCTJrzM0AQKsu_5k134s')::http_header
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

### Installing the BrainFuck TLE

Once the TLE is in place, install the BrainFuck extension:

```sql
-- Install the BrainFuck extension using the database package manager
SELECT dbdev.install('mansueli@brainfuck');
CREATE EXTENSION "mansueli@brainfuck"
    SCHEMA public -- You can change it here to something else
    VERSION '4.2.0';
```
*⚠️ Warning:* Installing this extension **may void your database's warranty.**

### Example Function Using BrainFuck

Here’s an example that uses the BrainFuck TLE to format a date:

```sql
CREATE OR REPLACE FUNCTION public.format_date(input_date timestamp with time zone)
RETURNS text
LANGUAGE plpgsql
IMMUTABLE STRICT
AS $function$
BEGIN
return brainfuck(',>,>,>,>,>,>,<<,>>>,>,>>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]<<<<<<<<<<[-]>>>>>>>>>>>[-]++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<[-]>>>>>>>>>>>[-]++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<[-]>>>>>>>>>>>[-]++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+++++++>[-]>>>>>>>>>>>[-]++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+++++++++>[-]>>>>>>>>>>>[-]++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+++++++++>[-]>>>>>>>>>>>[-]++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+++++++>[-]>>>>>>>>>>>[-]++++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+>[-]>>>>>>>>>>>[-]++++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+>[-]>>>>>>>>>>>[-]++++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+>[-]>>>>>>>>>>>[-]++++++++++++[-<<<<<<<<<<<++++++++++>>>>>>>>>>>]<<<<<<<<<<<+>[-]>[-]>[-][-]<<<<<<<<<<<<<<<<.>.>>>>>>>>>>>>>>[-][-]>>[-]++++[-<<++++++++++>>]<<+++++++.<<<<<<<<<<<<<<<<<.>.>>>>>>>>>>>>>>>>[-][-]>>[-]++++[-<<++++++++++>>]<<+++++++.<<<<<<<<<<<<<<<<<<<<<<.>.>.>.>>>>>>>>>>>>>>>>>>>>[.>]',input_date::text);
END;
$function$
```

## The Future of SupaBrain

The journey of SupaBrain has just begun. Some exciting ideas for future enhancements include:

- **AI-Powered Code Generation:**  
  Imagine a world where ChatGPT refuses to generate BrainFuck code—simply citing ethical concerns. The irony isn’t lost on us.
  
- **Multi-threaded BrainFuck Execution:**  
  While PostgreSQL isn’t ready for it yet, we dream of a day when BrainFuck can run concurrently across multiple threads.
  
- **BrainFuck ORM:**  
  Stay tuned for our BrainFuck Object-Relational Mapping tool. It’s coming soon—just as soon as we figure out how to map tables to tape memory.

## Conclusion: The Feature You Never Knew You Didn’t Need

SupaBrain transforms your database experience into something frustratingly unique. Whether it’s for a quirky hiring filter, or simply to prove that **just because you can, doesn’t mean you should**, SupaBrain has a place in every developer’s toolkit. So, why not give it a try? Or don’t—we promise not to judge.

Happy coding (or decoding)!