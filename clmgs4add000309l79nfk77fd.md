---
title: "Migrating from MongoDB to Supabase with PostgreSQL"
seoTitle: "Migrating from MongoDB to Supabase with Postgres"
seoDescription: "Learn how to migrate seamlessly from MongoDB to Supabase with PostgreSQL in this step-by-step guide. Enhance your database's scalability and performance."
datePublished: Tue Sep 12 2023 20:42:25 GMT+0000 (Coordinated Universal Time)
cuid: clmgs4add000309l79nfk77fd
slug: migrating-from-mongodb-to-supabase-with-postgresql
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1694550893974/f679a36c-45a4-455f-bb57-60c26aea555b.png
tags: postgresql, mongodb, databases, migration, data-migration

---

In the realm of database management, making informed decisions can significantly impact your application's performance and scalability. This comprehensive guide will offer you one possible way of migrating from MongoDB to [Supabase](http://supabase.com/) with PostgreSQL. There's an ongoing effort to improve this migration to make it seamless.

### Why Migrate from MongoDB to Supabase with PostgreSQL?

MongoDB has been a go-to choice for its flexibility, but as your project matures, scalability and complex querying challenges may arise. Supabase, built on PostgreSQL, combines the best of both worlds—a flexible NoSQL-style database with the reliability and performance of PostgreSQL. This migration opens up new possibilities for your application, making it a vital step for your project's growth.

## Understanding the Migration Process

### Why Choose Supabase and PostgreSQL for Your Migration?

To kick off your migration journey, let's delve into why Supabase and PostgreSQL are the great choices:

* **Scalability**: Supabase, powered by PostgreSQL, offers superb scalability, making it a fit for projects of all sizes.
    
* **Performance**: PostgreSQL is renowned for its speed and efficiency, ensuring smooth application operation even as your data grows.
    
* **Ease of Use**: Supabase simplifies database management with an intuitive interface, catering to developers of all skill levels.
    

### Planning the Migration Process

Before diving into the technical aspects, meticulous planning is paramount for a seamless migration. Considerations like data mapping, schema design, and data transformation require careful attention. As discussed in our previous article on ["Exploring Data Relationships with Supabase and PostgreSQL"](https://blog.mansueli.com/exploring-data-relationships-with-supabase-and-postgresql), understanding your data's structure is a crucial initial step.

## Running the Migration

You can run the migration process using this Colab notebook I've prepared for your convenience. It provides step-by-step instructions to ensure a smooth transition from MongoDB to Supabase with PostgreSQL. Below, we'll go through the steps to set it up yourself:

## Preparing Your Environment

### Installing the Required Python Libraries

Let's begin by installing the essential Python libraries for your migration journey:

```python
pip install mongo
pip install psycopg2
```

## Setting Up Connection URIs

To begin the migration process, it's essential to configure the connection URIs for both MongoDB and Supabase. Make sure you have your credentials ready and set the environment variables as shown below:

```python
# Source DB variables:
%env supabase_uri=postgresql://postgres:password@db.xxasaxx.supabase.co:5432/
%env mongo_uri=mongodb+srv://nacho:password@cluster001.jjj.mongodb.net/?retryWrites=true&w=majority
%env mongo_db=sample_mflix
```

## Running the Migration Manually

### Mapping Data Types

Mapping data types from MongoDB to PostgreSQL is a crucial step in the migration process. The provided script handles these conversions intelligently based on the Python types encountered in your MongoDB data. For example, it recognizes `ObjectId` and correctly maps it to the equivalent PostgreSQL data type. This ensures that your data retains its integrity throughout the migration process.

Here's how data types are mapped in the script:

```python
# Mapping MongoDB types to PostgreSQL types
SQL_DATA_TYPE = {
    "string": "TEXT",
    "ObjectId": "TEXT",
    "datetime": "TIMESTAMP WITH TIME ZONE",
    "int": "INT",
    "list": "JSONB",
    "dict": "JSONB",
    "bool": "Boolean",
    "float": "NUMERIC",
    "default": "TEXT",
}
```

### Creating PostgreSQL Tables

Creating the necessary tables in PostgreSQL is seamlessly handled by the provided script. If a table doesn't already exist for a MongoDB collection, the script creates one. Additionally, it checks for existing tables to avoid duplicates. This ensures that your data is organized efficiently in the PostgreSQL database, making it ready for further use.

In the script, you have the flexibility to specify the target MongoDB database using the `mongo_db_manual` variable. If left empty, the script will run for all databases in MongoDB.

For databases with more resources, consider including `collection.find(no_cursor_timeout=True)` in the code to prevent cursor timeouts during the migration.

## Migration code

Here's the migration code that establishes connections, maps data types, and creates PostgreSQL tables:

```python
from bson.decimal128 import Decimal128
import pymongo
import psycopg2
from psycopg2.extensions import AsIs
import json
from datetime import datetime
from psycopg2 import sql, extensions, connect, Error
from bson import ObjectId
import os

mongo_url = os.environ['mongo_uri']
supabase_url = os.environ['supabase_uri']

class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, ObjectId):
          return str(obj)
        if isinstance(obj, datetime):
          return obj.isoformat()
        if isinstance(obj, Decimal128):
          return str(obj)
        if isinstance(obj, complex):
            return [obj.real, obj.imag]
        return json.JSONEncoder.default(self, obj)


psycopg2.extensions.register_adapter(Decimal128, lambda val: AsIs(str(val.to_decimal())))

# Connect to MongoDB
mongo_client = pymongo.MongoClient(mongo_url)
# Connect to PostgreSQL
pg_conn = connect(supabase_url)
pg_conn.set_isolation_level(extensions.ISOLATION_LEVEL_AUTOCOMMIT)
pg_cur = pg_conn.cursor()

# Mapping MongoDB types to PostgreSQL types
SQL_DATA_TYPE = {
  "str": "TEXT",
  "ObjectId": "TEXT",
  "datetime.datetime": "TIMESTAMP WITH TIME ZONE",
  "datetime": "TIMESTAMP WITH TIME ZONE",
  "int": "INT",
  "list": "JSONB",
  "dict": "JSONB",
  "bool": "Boolean",
  "float": "NUMERIC",
  "default": "TEXT",
  "NoneType":"TEXT",
  "Decimal128":"NUMERIC",
}

# Store the type of each field
field_types = {}

# Get the list of database names from MongoDB
mongo_db_manual = os.environ['mongo_db']
mongo_db_names = []
if(len(mongo_db_manual)>0):
  mongo_db_names.append(mongo_db_manual)
else:
  mongo_db_names = mongo_client.list_database_names()

# Iterate over all MongoDB databases
for db_name in mongo_db_names:
    print("Starting to migrate :"+ str(db_name))
    mongo_db = mongo_client[db_name]

    # Iterate over all collections in the current database
    for collection_name in mongo_db.list_collection_names():
        # Skip system collections
        if collection_name.startswith("system."):
            continue

        collection = mongo_db[collection_name]
        # Create table in PostgreSQL if it doesn't exist
        pg_cur.execute(sql.SQL("CREATE TABLE IF NOT EXISTS {} ()").format(
            sql.Identifier(collection_name)))

        # Iterate over all documents in the collection
        cursor = collection.find()
        for document in cursor:
            # For each document, build a list of fields and a list of values
            fields = []
            values = []
            for field, value in document.items():
                # Determine PostgreSQL type based on Python type
                if isinstance(value, ObjectId):
                    pg_type = SQL_DATA_TYPE["ObjectId"]
                    value = str(value)
                else:
                    pg_type = SQL_DATA_TYPE.get(type(value).__name__, SQL_DATA_TYPE["default"])

                # Add type suffix to field name if a new type is encountered
                field_with_type = field
                if field in field_types:
                    if type(value).__name__ not in field_types[field]:
                        field_types[field].add(type(value).__name__)
                        field_with_type = f"{field}_{type(value).__name__}"
                else:
                    field_types[field] = {type(value).__name__}

                # Add column in PostgreSQL if it doesn't exist
                try:
                    pg_cur.execute(sql.SQL("ALTER TABLE {} ADD COLUMN {} {}").format(
                        sql.Identifier(collection_name),
                        sql.Identifier(field_with_type),
                        sql.SQL(pg_type)))
                except Error:
                    pass  # Column already exists, no action needed

                # Add field and value to the lists
                fields.append(sql.Identifier(field_with_type))
                if isinstance(value, list) or isinstance(value, dict):
                    value = json.dumps(value, cls=CustomEncoder)
                values.append(value)

            # Insert data into PostgreSQL
            pg_cur.execute(sql.SQL("INSERT INTO {} ({}) VALUES ({})").format(
                sql.Identifier(collection_name),
                sql.SQL(', ').join(fields),
                sql.SQL(', ').join(sql.Placeholder() * len(values))),
                values)

pg_cur.close()
pg_conn.close()
```

This script simplifies the migration process by automating many of the essential tasks.

## Data Transformation and Beyond

### Making the Transition

With your connection set up and tables created, it's time to dive into the heart of the migration process - transforming MongoDB documents into PostgreSQL rows. This step is pivotal for a successful migration and ensuring that your data remains intact.

**The Data Transformation Journey**

The journey of transforming your data from MongoDB to PostgreSQL is where the magic happens. MongoDB and PostgreSQL have distinct data models, which means a meticulous transformation process is crucial. We won't just guide you through this journey; we'll take you on a hands-on tour with detailed code samples and explanations.

**Code in Action**

Our approach is all about clarity and understanding. We provide code samples that vividly showcase the transformation of MongoDB data into a PostgreSQL-compatible format. Each sample is accompanied by a comprehensive explanation, ensuring that you not only get the job done but also understand the underlying nuances and intricacies.

**Exploring Alternatives**

While our primary focus has been on migrating from MongoDB to PostgreSQL, it's always a good practice to explore alternative solutions. One compelling alternative is [FerretDB](https://www.ferretdb.io/), a MongoDB-like database using PostgreSQL that can be hosted within the Supabase ecosystem. Depending on your specific requirements and preferences, this might be an excellent choice to consider.

### Handling Challenges

Migration projects often encounter unexpected challenges. It's essential to acknowledge that no migration process is entirely free of hurdles. While we've covered a broad spectrum of challenges and provided solutions, unique scenarios may require specialized attention. In such cases, it's advisable to seek professional guidance to ensure a seamless migration experience.

## Post-Migration Considerations

With your data successfully migrated to PostgreSQL through Supabase, let's shift our focus to what comes next - ensuring your database performs optimally and reliably.

### Data Validation and Optimization

After completing the migration, your first priority should be data validation and testing. It's not just about getting your data into the new system; it's about making sure it made the transition accurately and retains its integrity. Learn about best practices and available tools for this crucial step. We'll guide you through the process, leaving no room for uncertainty.

**Unleashing PostgreSQL's Power**

To fully harness the capabilities of PostgreSQL, it's essential to optimize your database's performance. This section is a treasure trove of insights into key areas like indexing, query optimization, and other relevant topics. Discover how to fine-tune your database for optimal speed and efficiency, ensuring that your application runs smoothly.

## Conclusion

In conclusion, migrating from MongoDB to Supabase with PostgreSQL can be a transformative step for your application. We've covered crucial stages, from understanding the advantages of this transition to handling post-migration considerations. Please note that this is an initial effort migrating from live connections and is not performant at this time. I am also working on an alternative migration guide from `mongodump` to Postgres in this [GitHub repo](https://github.com/mansueli/mongo-2-postgres/t). Contributions are very welcome.

### Additional Resources

To further assist you in your migration journey, here are some additional resources:

* [Supabase Documentation](https://supabase.com/docs): Explore Supabase's official documentation for in-depth information and guidance.
    
* [PostgreSQL Official Documentation](https://www.postgresql.org/docs/): Dive into the official PostgreSQL documentation to expand your knowledge of this powerful database system.
    
* [GitHub Repository with Migration Code Samples](https://github.com/mansueli/Supa-Migrate/blob/main/mongo2supabase.ipynb): Access our GitHub repository, where you can find detailed code samples and resources related to the migration process.
    

Feel free to explore these resources as you embark on your journey to a more efficient and robust database solution. Happy migration!