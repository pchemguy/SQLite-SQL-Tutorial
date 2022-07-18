---
layout: default
title: Fixed-Width Values
nav_order: 3
parent: String Manipulation
permalink: /strings/black-boxed-queries
---

### Context

Consider a tree structure representing a hierarchical system of categories, and this structure needs to be stored in a relational database. While the classical relational model does not mesh well with hierarchical data,  there are several approaches to marrying these concepts (see [References](#References) for further information).

In the Materialized/Enumerated Paths, each node saved as a row in the *nodes* table stores information about its absolute path. There are several approaches to encoding this path, which may or may not include the node itself. Let us assume that each node is assigned an ID consisting of eight randomly selected alphanumeric ASCII characters. An analog of file system path constructed from a sequence of ancestor IDs can act as the node path. Furthermore, because node ID has a fixed length, a "path separator" is not necessary, for example: 

~~~json
{"BE0A8514": "afc2e40a40CF97B4704BA7F4F4CE4D5BF5A2A524"}
~~~

where the key term is node ASCII ID, and the value term is node path/prefix, not including the node itself. The task is to convert the latter to a sequence of ancestor IDs.

### Splitting fixed-width value strings

Like in the case of [DSV][] strings, the following query iscontext-specific, but the notes below should help adapt it to other needs.

<a name="FWV-Query"></a>
~~~sql
WITH
    prefixes(ascii_id, prefix) AS (
        VALUES
            ('0FDAF2C8', 'afc2e40a40CF97B482A35587'),
            ('BE0A8514', 'afc2e40a40CF97B4704BA7F4F4CE4D5BF5A2A524')
    ),
    id_sizes AS (SELECT length(ascii_id) AS id_size FROM prefixes LIMIT 1),
    positions AS (
        SELECT
            ascii_id, prefix, "key" * id_size + 1 AS position
        FROM
            id_sizes, prefixes,
            json_each('[' || replace(hex(zeroblob(length(prefix)/id_size - 1)), '00', '0,') || '0]')
    ),
    ascii_ids AS (
        SELECT ps.*, substr(prefix, position, id_size) AS asc_ascii_id
        FROM positions AS ps, id_sizes
        ORDER BY ascii_id, position
    ),
    json_prefixes AS (
        SELECT ascii_id, prefix, json_group_array(asc_ascii_id) AS prefix_json
        FROM ascii_ids
        GROUP BY ascii_id
    )
SELECT * FROM json_prefixes;
~~~

 1. ID length is necessary to split the prefix. This value may be provided via a dedicated query parameter or hardcoded. *id_sizes* uses a third option: because this query expects both node id and prefix, it grabs node ID from the first input pair and takes its length.
 2. *positions* query is possibly a bit over-engineered way of creating a table containing offsets of different *prefix* components to be used by the *substr* function. To start, the query determines length(*prefix*) / length(*node ID*) = *IDs_in_prefix* ratio used to produce a dummy JSON array of the same length. First, the *hex* and *zeroblob* functions produce a zero-field string template. Then, the *replace* function inserts JSON element separators (commas) into the template. Because the *hex/zeroblob* pair produces a doubled length string, _replace_ swaps every two zeros with a single zero. There is no comma after the last "0", so *IDs_in_prefix* is reduced by one, and the last zero prefixes the closing bracket. Finally, the *json_each* table-valued function splits this template and returns a table with the "key" column containing the offsets of JSON array elements, and "key" x id_size +  1 can be used as offsets for *substr*.
 3. *ascii_ids* uses *substr* to generate rows containing *prefix* elements labeled with both node ID and the element position in the original string.
4. *json_prefixes* collapses rows belonging to the same node ID (the *ascii_ids* is sorted on *ascii_id* and *position*), yielding the final result.

Query output:

| ascii_id | prefix                                   | prefix_json                                              |
|----------|------------------------------------------|----------------------------------------------------------|
| 0FDAF2C8 | afc2e40a40CF97B482A35587                 | ["afc2e40a","40CF97B4","82A35587"]                       |
| BE0A8514 | afc2e40a40CF97B4704BA7F4F4CE4D5BF5A2A524 | ["afc2e40a","40CF97B4","704BA7F4","F4CE4D5B","F5A2A524"] |

The *prefixes* query, as before, defines mock input data and can be extended

~~~sql
    prefixes AS (
        SELECT "key" AS ascii_id, value AS prefix FROM json_each(
            '{' ||
                '"0FDAF2C8": "afc2e40a40CF97B482A35587",' ||
                '"BE0A8514": "afc2e40a40CF97B4704BA7F4F4CE4D5BF5A2A524"' ||
            '}'
        )
    ),
~~~

to switch to the JSON-based input format:

~~~json
{"ascii_id_1": "prefix_1", "ascii_id_2": "prefix_2",...}
~~~




<a name="References"></a>
### References

1. [Joe Celko's Trees and Hierarchies in SQL for Smarties, 2nd edition][Celko's Trees]
2. [SQL Design Patterns: The Expert Guide to SQL Programming][Tropashko]
3. [Nested Sets and Materialized Path SQL Trees][NS-MP]
4. [Storing trees in RDBMS][Kolesnikova]
5. [Django-Treebeard tree library for Django Web Framework][django-treebeard]
6. [PostgreSQL tree module ltree][PostgreSQL ltree]


<!-- References -->

[Celko's Trees]: https://sciencedirect.com/book/9780123877338
[Tropashko]: https://vadimtropashko.wordpress.com/%22sql-design-patterns%22-book/about
[NS-MP]: http://rampant-books.com/art_vadim_nested_sets_sql_trees.htm
[django-treebeard]: https://django-treebeard.readthedocs.io
[PostgreSQL ltree]: https://www.postgresql.org/docs/current/ltree.html
[Kolesnikova]: https://bitworks.software/en/2017-10-20-storing-trees-in-rdbms.html
[DSV]: /strings/split-dsv#DSV-Query