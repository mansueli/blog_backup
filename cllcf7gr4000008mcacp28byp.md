---
title: "How to Boost Supabase: A Guide to Using Postgres Foreign Data Wrappers"
seoTitle: "How to Boost Supabase Reliability: Guide for Foreign Data Wrappers"
seoDescription: "Learn how to enhance Supabase project reliability by leveraging Foreign Data Wrappers. Discover seamless integration for improved robustness."
datePublished: Tue Aug 15 2023 14:50:11 GMT+0000 (Coordinated Universal Time)
cuid: cllcf7gr4000008mcacp28byp
slug: how-to-boost-supabase-a-guide-to-using-postgres-foreign-data-wrappers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1692110853459/cb4bbb06-3e08-4a8d-94b8-431086b95489.png
tags: postgresql, databases, sql, supabase, fdw

---

Welcome to this comprehensive guide on enhancing the reliability and isolation of your Supabase project using the power of Foreign Data Wrappers (FDW). In this blog post, we will delve into the strategic implementation of FDWs to connect your project with an independent queue system. By adopting this approach, your Supabase project gains an additional layer of resilience, ensuring that even if your primary system encounters issues, the queue continues to operate seamlessly. This exploration promises to equip you with the knowledge to fortify your application's robustness. In a previous post, we explored [how to create a pseudo FDW to MySQL](https://blog.mansueli.com/mysql-foreign-data-wrapper-supabase-postgresql-edge-functions) using an Edge Function as a bridge, now we'll explore using proper Postgres FDW.

### **Prerequisites for Implementing FDWs**

To embark on this journey, it's essential to have the following prerequisites in place.

1. **Main Supabase Project:** You should have your primary Supabase project set up and operational. This serves as the foundation upon which we'll build the enhanced system.
    
2. **Foreign Supabase Project:** Similarly, your separate foreign Supabase project needs to be ready. This distinct database will be integrated using the Foreign Data Wrappers approach.
    

Before proceeding, ensure these prerequisites are aligned to make the most of this guide.

**Exploring the Process:** The process we're about to undertake involves creating a bridge between your main Supabase project and the foreign Supabase project through the magic of Foreign Data Wrappers. This bridge facilitates communication and data sharing between the projects, offering enhanced reliability and isolation.

### **Create a User in the Foreign Database:**

In this crucial step, we lay the foundation for secure communication. By creating a dedicated user 'foreign\_user' and granting it specific privileges, we establish the necessary credentials to bridge the databases. The use of BYPASSRLS ensures controlled data access.

```sql
CREATE USER foreign_user WITH PASSWORD 'password' BYPASSRLS;

GRANT SELECT, INSERT, UPDATE, DELETE 
ON ALL TABLES IN SCHEMA public 
TO foreign_user;

GRANT USAGE, SELECT 
ON ALL SEQUENCES IN SCHEMA public 
TO foreign_user;
```

### **Configuration in the Main Database:**

This step involves configuring your main Supabase database to communicate with the foreign server. We set up a new schema 'queue' to house the imported foreign data. Additionally, we establish a foreign server named 'foreign\_server,' specifying connection details like host, port, and dbname. The user mapping ensures secure authentication, while the IMPORT FOREIGN SCHEMA command integrates the foreign schema into your main database.

```sql
CREATE EXTENSION postgres_fdw;
CREATE SCHEMA queue;

ALTER DEFAULT PRIVILEGES IN SCHEMA queue
GRANT ALL ON TABLES TO postgres;

CREATE SERVER foreign_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'db.foreign.supabase.co', port '5432', dbname 'postgres');

CREATE USER MAPPING FOR postgres
    SERVER foreign_server
    OPTIONS (user 'foreign_user', password 'password');

IMPORT FOREIGN SCHEMA public
    FROM SERVER foreign_server
    INTO queue;

ALTER SERVER foreign_server 
    OPTIONS (fetch_size '50000');
```

**Explaining fetch\_size Adjustment:** The `ALTER SERVER` command with the `fetch_size` option plays a vital role in optimizing data retrieval. By setting the fetch size to '50000,' we dictate how many rows of data the server fetches in a single request. Larger fetch sizes can enhance data retrieval efficiency, especially when dealing with substantial datasets.

**Enhancing Queue Reliability:** This approach not only boosts the reliability of your main Supabase project but also increases the robustness of your queue system. To understand more about creating a queue system using Supabase and PostgreSQL, refer to the article [Building a Queue System with Supabase and PostgreSQL](https://blog.mansueli.com/building-a-queue-system-with-supabase-and-postgresql).

### **Conclusion**

By meticulously following these steps, you pave the way for a robust, interconnected Supabase ecosystem. The integration of Foreign Data Wrappers fosters reliability, ensuring that the queue system remains resilient even in the face of challenges.

Don't hesitate to explore the [Supabase documentation](https://supabase.com/docs/guides/database/extensions), which offers a wealth of knowledge on database extensions and their applications.