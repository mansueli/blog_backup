---
title: "Exploring Data Relationships with Supabase and PostgreSQL"
seoTitle: "Unleash Data Management with Supabase-JS using JOIN Queries"
seoDescription: "Discover the power of data relationships, foreign keys, and Supabase-JS for dynamic database management."
datePublished: Tue Aug 08 2023 13:51:11 GMT+0000 (Coordinated Universal Time)
cuid: cll2d0mb1001c09mce6pf7zuf
slug: exploring-data-relationships-with-supabase-and-postgresql
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691501802483/645c7277-621c-413f-bc08-64d742430652.png
tags: postgresql, databases, sql, supabase

---

Discover Effective Data Relationship Management with Supabase Join and Inner Join Techniques.

Supabase, the potent open-source platform, brings you real-time and secure backend-as-a-service capabilities, leveraging the power of PostgreSQL. Uncover the art of managing data relationships effectively using Supabase and PostgreSQL.

## Utilizing Foreign Keys to Establish Table Links

The foundation of dynamic and efficient applications lies in robust and interconnected databases. One crucial technique for achieving this is harnessing the potential of foreign keys to establish relationships between database tables. Let's delve into how you can leverage foreign keys for linking tables and executing advanced JOIN queries.

### Creating Tables for Demonstrating JOIN Queries

Before we plunge into data relationships, let's kick things off by setting up the required database tables. Our example involves five distinct tables: `books`, `publishers`, `translations`, `teachers`, and `courses`. Each table serves a unique purpose in our application. For instance, the `books` table stores book details, the `publishers` table holds publisher information, and the `translations` table tracks book translations.

```sql
CREATE TABLE books (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL
);

CREATE TABLE publishers (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    book_id INTEGER,
    name TEXT NOT NULL,
    published_date DATE,
    FOREIGN KEY (book_id) REFERENCES books(id)
);

CREATE TABLE translations (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    book_id INTEGER,
    language TEXT NOT NULL,
    translator TEXT,
    FOREIGN KEY (book_id) REFERENCES books(id)
);
```

Here's a glimpse of example insert statements:

```sql
INSERT INTO books (title, author) VALUES
    ('The Great Gatsby', 'F. Scott Fitzgerald'),
    ('To Kill a Mockingbird', 'Harper Lee'),
    ('1984', 'George Orwell'),
    ('Pride and Prejudice', 'Jane Austen'),
    ('The Hobbit', 'J.R.R. Tolkien'),
    ('Harry Potter and the Sorcerer''s Stone', 'J.K. Rowling'),
    ('The Catcher in the Rye', 'J.D. Salinger'),
    ('Lord of the Flies', 'William Golding'),
    ('The Chronicles of Narnia', 'C.S. Lewis'),
    ('Brave New World', 'Aldous Huxley');
INSERT INTO publishers (book_id, name, published_date) VALUES
    (1, 'Scribner', '1925-04-10'),
    (2, 'HarperCollins', '1960-07-11'),
    (3, 'Secker & Warburg', '1949-06-08'),
    (4, 'T. Egerton, Whitehall', '1813-01-28'),
    (5, 'Allen & Unwin', '1937-09-21'),
    (6, 'Bloomsbury', '1997-06-26'),
    (7, 'Little, Brown', '1951-07-16'),
    (8, 'Faber and Faber', '1954-09-17'),
    (9, 'Geoffrey Bles', '1950-10-16'),
    (10, 'Chatto & Windus', '1932-06-23');
INSERT INTO translations (book_id, language, translator) VALUES
    (1, 'French', 'Maurice Ravel'),
    (1, 'German', 'Franz Kafka'),
    (2, 'Spanish', 'Gabriel García Márquez'),
    (2, 'Italian', 'Italo Calvino'),
    (3, 'Russian', 'Yevgeny Zamyatin'),
    (3, 'Chinese', 'Lu Xun'),
    (4, 'Japanese', 'Haruki Murakami'),
    (4, 'Korean', 'Shin Kyung-sook'),
    (5, 'Spanish', 'Jorge Luis Borges'),
    (5, 'Arabic', 'Naguib Mahfouz');
```

### Performing JOIN Queries Using Supabase-JS

Unlock the ability to combine data from multiple tables with inner joins, using the Supabase-JS JavaScript library. This user-friendly tool simplifies database querying and inner join operations, ensuring a seamless experience.

To seamlessly perform an inner join using Supabase-JS, leverage the `select` method while specifying the desired columns for retrieval. Consult the [Supabase documentation](https://supabase.io/docs/guides/database/select) for comprehensive guidance on working with Supabase-JS.

With the groundwork laid for data relationships through foreign keys, let's delve into practical examples of executing JOIN queries using Supabase-JS.

### Basic JOIN Operation: Connecting Books and Publishers Tables

Here's a fundamental JOIN operation between the `books` and `publishers` tables with this JavaScript code snippet:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692645687933/070a42a6-366f-4a5b-975f-985960fccac1.png align="center")

```javascript
const { data, error } = await supabase  
      .from('books')
      .select('*,publishers(*)');
/* Example response:
[
  {
    "id": 1,
    "title": "The Great Gatsby",
    "author": "F. Scott Fitzgerald",
    "publishers": [
      {
        "id": 1,
        "book_id": 1,
        "name": "Scribner",
        "published_date": "1925-04-10"
      }
    ]
  },
  {
    "id": 2,
    "title": "To Kill a Mockingbird",
    "author": "Harper Lee",
    "publishers": [
      {
        "id": 2,
        "book_id": 2,
        "name": "HarperCollins",
        "published_date": "1960-07-11"
      }
    ]
  }
]
```

The outcome of this query presents a unified view of books alongside their corresponding publishers.

### Fine-Tuning Queries with Foreign Table Filters

Refine your queries by filtering based on attributes of the foreign table. Here's an illustrative example:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692645704800/5c700d73-95ea-4819-b21a-5addb64da494.png align="center")

```javascript
const { data, error } = await supabase
    .from('books')
    .select('*,publishers!inner(name,published_date)')
    .eq('publishers.name', 'Bloomsbury');

/* Example response:
[
  {
    "id": 6,
    "title": "Harry Potter and the Sorcerer's Stone",
    "author": "J.K. Rowling",
    "publishers": [
      {
        "name": "Bloomsbury",
        "published_date": "1997-06-26"
      }
    ]
  }
]
```

This query retrieves books published by "Bloomsbury" along with their corresponding details.

### Uncover Insights with Foreign Table Count

You can also use JOINs to retrieve aggregate data from the foreign table. This example showcases counting translations for each book:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692646263183/ac43ff37-cd79-4caf-ae01-a9970d938777.png align="center")

```javascript

const { data, error } = await supabase
  .from('books')
  .select(`*, translations(count)`)
/* Example response:
[
  {
    "id": 1,
    "title": "The Great Gatsby",
    "author": "F. Scott Fitzgerald",
    "translations": [
      {
        "count": 2
      }
    ]
  },
  {
    "id": 2,
    "title": "To Kill a Mockingbird",
    "author": "Harper Lee",
    "translations": [
      {
        "count": 2
      }
    ]
  },
  {
    "id": 3,
    "title": "1984",
    "author": "George Orwell",
    "translations": [
      {
        "count": 2
      }
    ]
  }
]
```

This query returns the books along with the count of translations for each.

### Anti-Joins by checking for nulls

It is possible to isolate the rows that are shared with the following approach:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692646155506/836a53fb-8b2a-4d92-a3ef-0f997aed3fb5.png align="center")

```sql
supabase
  .from("books")
  .select("*,translations()")
  .is("translations", null)
```

### Forging Relationships between Unrelated Tables

Discover a surprising capability: establishing connections between seemingly unrelated tables via the parent table:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692645783952/f3e8cc43-c176-4425-b02f-bf42c6fc53c3.png align="center")

```javascript
const { data, error } = await supabase
    .from('books').select('publishers(*),translations(*)');
/* Example response:
[
  {
    "publishers": [
      {
        "id": 1,
        "book_id": 1,
        "name": "Scribner",
        "published_date": "1925-04-10"
      }
    ],
    "translations": [
      {
        "id": 1,
        "book_id": 1,
        "language": "French",
        "translator": "Maurice Ravel"
      },
      {
        "id": 2,
        "book_id": 1,
        "language": "German",
        "translator": "Franz Kafka"
      }
    ]
  },
  {
    "publishers": [
      {
        "id": 2,
        "book_id": 2,
        "name": "HarperCollins",
        "published_date": "1960-07-11"
      }
    ],
    "translations": [
      {
        "id": 3,
        "book_id": 2,
        "language": "Spanish",
        "translator": "Gabriel García Márquez"
      },
      {
        "id": 4,
        "book_id": 2,
        "language": "Italian",
        "translator": "Italo Calvino"
      }
    ]
  }
]
```

In this instance, we amalgamate data from the `publishers` and `translations` tables based on the intermediary `books` table, even without a direct link.

## Exploring the Realm of Computed Relationships

Having mastered foreign keys and JOIN queries, let's elevate our understanding by venturing into computed relationships—a potent feature bestowed by Supabase and PostgreSQL. Computed relationships empower us to bridge gaps between tables and derive meaningful connections among data points.

### Setting the Stage with Example Tables

To illustrate computed relationships, we'll create example tables `planets` and `moons`, showcasing how this advanced technique can enhance our data management.

```sql
-- Computed relationships:
CREATE TABLE planets (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE moons (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    planet_id INTEGER,
    name TEXT NOT NULL
);

-- Relationship function (many to one):
CREATE FUNCTION planets(moons) RETURNS SETOF moons ROWS 1 AS $$
  SELECT * FROM moons WHERE id = $1.planet_id
$$ STABLE LANGUAGE SQL;
```

Let's add some rows to these tables to test out our computed relationships:

```sql
INSERT INTO planets (name) VALUES
    ('Mercury'),
    ('Venus'),
    ('Earth'),
    ('Mars'),
    ('Jupiter'),
    ('Saturn'),
    ('Uranus'),
    ('Neptune');
INSERT INTO moons (planet_id, name) VALUES
    (3, 'Moon'),
    (4, 'Phobos'),
    (4, 'Deimos'),
    (5, 'Io'),
    (5, 'Europa'),
    (5, 'Ganymede'),
    (5, 'Callisto'),
    (6, 'Titan'),
    (6, 'Enceladus'),
    (7, 'Titania');
```

### Probing Computed Relationships

With the stage set, let's explore computed relationships using Supabase-JS. The following JavaScript snippet exemplifies querying a computed relationship between the `moons` and `planets` tables:

```javascript
const { data, error } = await supabase
    .from('moons').select('*,planets(*)');
/* Example response:
[
  {
    "id": 1,
    "planet_id": 3,
    "name": "Moon",
    "planets": {
      "id": 3,
      "planet_id": 4,
      "name": "Deimos"
    }
  },
  {
    "id": 2,
    "planet_id": 4,
    "name": "Phobos",
    "planets": {
      "id": 4,
      "planet_id": 5,
      "name": "Io"
    }
  }
]
*/
```

The resulting response showcases the enhanced connections between moons and their parent planets.

### Conclusion

We demonstrated the potential of Supabase-JS and PostgreSQL, propelling data management to new heights. From rudimentary JOIN queries to sophisticated computed relationships, these tools metamorphose serverless application data management. Armed with these techniques, developers wield the power to craft adaptable, efficient, and interlinked systems tailored to diverse data relationship scenarios.

Embark on your journey with Supabase and PostgreSQL, exploring further features, optimization tactics, and real-world application scenarios. The realm of data management awaits—harness its might with Supabase!

Stay tuned for more illuminating insights and enriching content in forthcoming blog posts.