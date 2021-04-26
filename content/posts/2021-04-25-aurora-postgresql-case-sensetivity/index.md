---
layout: post
title: Case-sensitivity in Aurora PostgreSQL
date: '2021-04-25'
author: Brian
tags: 
- Aurora
- PostgreSQL
---

When moving from SQL Server to PostgreSQL a common issue is case sensitivity. PostgreSQL 12 adds support for nondeterministic collations. These are my notes from testing various scenarios in Aurora. In my opinion CITEXT is still the best option despite native support. 

## Create some test data

First, Create a new Case Sensitive and Case Insensitive collation. Technically these exist under other names, but I thought his was more clear in the examples. The important part is the **deterministic** flag. This is Unicode speak for compare the bytes (true) or compare the logical values (false).

```sql
CREATE COLLATION case_sensitive (provider = icu, locale = 'und-u-ks-level2', deterministic = true);
CREATE COLLATION case_insensitive (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
```

Then, I use the collations in a table definition.

```sql
CREATE TABLE superheros (
    name1 text COLLATE "case_sensitive",
    name2 text COLLATE "case_insensitive"
);
```

Finally, I insert some data with mixed case. 

```sql
INSERT INTO superheros(name1, name2) VALUES ('superman', 'superman');
INSERT INTO superheros(name1, name2) VALUES ('Superman', 'Superman');
INSERT INTO superheros(name1, name2) VALUES ('supergirl', 'supergirl');
INSERT INTO superheros(name1, name2) VALUES ('Supergirl', 'Supergirl');
INSERT INTO superheros(name1, name2) VALUES ('spiderman', 'spiderman');
INSERT INTO superheros(name1, name2) VALUES ('Spiderman', 'Spiderman');
```


## Look at the impact on comparisons 

First, search for data without the custom collation. Note only one row is returned.

```sql
SELECT * FROM superheros WHERE name1 = 'Spiderman';
```

And, again with the custom collation. Note both rows are returned. 

```sql
SELECT * FROM superheros WHERE name2 = 'Spiderman';
```

Alternatively, you could just specify the collation at the time of the search rather than defining the column with this collation. 

```sql
SELECT * FROM superheros WHERE name1 = 'Spiderman' COLLATE "case_insensitive"; 
```

Cool that seems to work as expected. 

# What about LIKE statements?

PSQL supports a LIKE for case-sensitive and ILIKE case-insensitive comparisons. This was true before v12 and nothing has changed. So, LIKE will return only 2 rows

```sql
SELECT * FROM superheros WHERE name1 LIKE 'Super%';
```

While, ILIKE will return 4 rows.

```sql
SELECT * FROM superheros WHERE name1 ILIKE 'Super%';
```

However, you cannot use either LIKE or ILIKE on name 2 with the non-deterministic (i.e. case-insensitive) collation. It throws an error. Ugh.

```sql
SELECT * FROM superheros WHERE name2 ILIKE 'Super%';
```

Therefore, you must explicitly specify a deterministic collation.

```sql
SELECT * FROM superheros WHERE name2 ILIKE 'Super%' COLLATE "case_sensitive";
```

That just feels clunky.


## What about CITEXT?


Recent versions of Aurora (event before v12) include the CITEXT extension. You can enable it for a database like this.

```sql
CREATE EXTENSION CITEXT;
```

And then you can create a field of type CITEXT.

```sql
CREATE TABLE test (test CITEXT);
```

Add some data

```sql
INSERT INTO test VALUES('Brian')
INSERT INTO test VALUES('brian')
INSERT INTO test VALUES('Ben');
INSERT INTO test VALUES('ben');
```

Then you can perform case-insensitive comparisons.

```sql
SELECT * FROM test WHERE test='Brian';
```

If also supports and use ILIKE as expected. 

```sql
SELECT * FROM test where test ILIKE 'B%'
```

Generally, that just feels more natural. So in the case of text fields, I still prefer the CITEXT type. 

## What about schema objects?

PostgreSQL uses lowercase schema names by default. For example, let's create a table with Pascal Case names common in SQL Server.

```sql
CREATE TABLE SideKicks (
    SuperHero text,
    "SideKick" text
);
```

Note that PostgreSQL ignored the case, except where I quote the fields. Therefore, all three of these work.

```sql
SELECT * FROM SideKicks;
SELECT * FROM sidekicks;
SELECT * FROM SIDEKICKS;
```

Also notice that the *SuperHero* is stored as *superhero* in the result. But, *SideKick* was left as is because of the double quotes. 

```sql
 superhero | SideKick 
-----------+----------
```

However, because I specified a specific case in the definition, I must query the table the same way. If I try to insert data without double quoting SideKick, PostgreSQL will assume lowercase and it will fail.

```sql
INSERT INTO SideKicks(SuperHero, SideKick) VALUES ('Batman', 'Robin');
ERROR:  column "sidekick" of relation "sidekicks" does not exist
```

If I specify a schema object with a specific case, I must always refer to that object with quotes.

```sql
INSERT INTO SideKicks(SuperHero, "SideKick") VALUES ('Batman', 'Robin');
```

Therefore, it's probably best to never use quotes. Some visual design tools always add the quotes so be careful of these. If you are using one of these tools try to always use lowercase schema object names.  