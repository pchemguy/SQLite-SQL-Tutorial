---
layout: default
title: Surrogate Variables
nav_order: 3
parent: Design Patterns
permalink: /patterns/variables
---

Let us rewrite the previous [query](/patterns/split-dsv#DSV-Query) using common tables expressions (CTEs):

~~~sql
WITH
    folders AS (
        SELECT "terms"."key" AS term_id, "terms"."value" AS term
        FROM json_each('["' || replace(trim('C:/Winows/System32/drivers/etc/', '/'), '/', '", "') || '"]') AS terms
    )
SELECT * FROM folders;
~~~

Suppose we wish to turn it into a library query (as discussed [earlier]( /patterns/decoupling-sql)), which should split a generic string. For now, we ignore the potential presence of certain characters, such as backslashes and double quotes within the terms. Query interface:

  - input - the string to be split and the delimiter;
  - output - a list of terms.

While we can parameterize the query above, let us, first, rewrite it as follows:

~~~sql
WITH
    delimiters(delimiter) AS (VALUES ('/')),
    strings(string) AS (VALUES ('C:/Winows/System32/drivers/etc/')),
    folders AS (
        SELECT "terms"."key" AS term_id, "terms"."value" AS term
        FROM
            delimiters, strings,
            json_each('["' || replace(trim(string, delimiter), delimiter, '", "') || '"]') AS terms
    )
SELECT * FROM folders;
~~~

The *delimiters* query is an "immediate" query returning a single row and a single column *delimiter*. The *terms* table in the *folders* query is now joined with this *delimiters* query. The incurred performance penalty due to this join should be minimal, if not negligible. At the same time, the *delimiter* column can now act as a surrogate variable. The same consideration applies to the *strings* query. This query can now be parameterized:

~~~sql
WITH
    delimiters(delimiter) AS (VALUES (@delimiter)),
    strings(string) AS (VALUES (@string)),
    folders AS (
        SELECT "terms"."key" AS term_id, "terms"."value" AS term
        FROM
            delimiters, strings,
            json_each('["' || replace(trim(string, delimiter), delimiter, '", "') || '"]') AS terms
    )
SELECT * FROM folders;
~~~

Can we process multiple strings with a single query call? For example,

~~~sql
WITH
    delimiters(delimiter) AS (VALUES ('/')),
    strings(string) AS (
        VALUES
            ('C:/Winows/System32/drivers/etc/'),
            ('C:/Users/Public/Desktop')
    ),
    folders AS (
        SELECT "terms"."key" AS term_id, "terms"."value" AS term
        FROM
            delimiters, strings,
            json_each('["' || replace(trim(string, delimiter), delimiter, '", "') || '"]') AS terms
    )
SELECT * FROM folders;
~~~

This query works just fine and splits both strings:

| term_id | term     |
|---------|----------|
| 0       | C:       |
| 1       | Winows   |
| 2       | System32 |
| 3       | drivers  |
| 4       | etc      |
| 0       | C:       |
| 1       | Users    |
| 2       | Public   |
| 3       | Desktop  |

But we now need the means to label the terms with a string ID. For example, the following query:

<a name="DSV-Query"></a>
~~~sql
WITH
    delimiters(delimiter) AS (VALUES ('/')),
    strings(string_id, string) AS (
        VALUES
            ('abc', 'C:/Winows/System32/drivers/etc/'),
            ('def', 'C:/Users/Public/Desktop')
    ),
    folders AS (
        SELECT string_id, "terms"."key" AS term_id, "terms"."value" AS term
        FROM
            delimiters, strings,
            json_each('["' || replace(trim(string, delimiter), delimiter, '", "') || '"]') AS terms
    )
SELECT * FROM folders;
~~~

produces properly labeled output:

| string_id | term_id | term     |
|-----------|---------|----------|
| abc       | 0       | C:       |
| abc       | 1       | Winows   |
| abc       | 2       | System32 |
| abc       | 3       | drivers  |
| abc       | 4       | etc      |
| def       | 0       | C:       |
| def       | 1       | Users    |
| def       | 2       | Public   |
| def       | 3       | Desktop  |

Can we make a parameterized library query if we want to split an arbitrary number of strings? Each input string requires a dedicated line in the *strings* block, and each such line needs two distinct query parameters. In other words, both the query code and the calling signature depend on the number of strings to be split, so a different approach is required.
