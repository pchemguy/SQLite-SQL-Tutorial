---
layout: default
title: SELECT Ancestors
nav_order: 4
parent: Materialized Paths
permalink: /mat-paths/select-asc
---

Retrieving node ancestors is another standard operation. Given a JSON array of category paths,

~~~json
["tcl/compat/zlib1/", "tcl/pkgs/thread2.8.7/tcl/cmdsrv/"]
~~~

produce the list of ancestors, such as

| ascii_id | asc_path                  |
|----------|---------------------------|
| 0FDAF2C8 | tcl/                      |
| 0FDAF2C8 | tcl/compat/               |
| BE0A8514 | tcl/                      |
| BE0A8514 | tcl/pkgs/                 |
| BE0A8514 | tcl/pkgs/thread2.8.7/     |
| BE0A8514 | tcl/pkgs/thread2.8.7/tcl/ |

Here, the first column labels individual terms with the IDs of input nodes, and the other column contains ancestor paths. Two features - recursive CTEs and the *json_tree* recursive table-valued function - can facilitate this conversion. The query presented below uses the latter. Consider the following base query

~~~sql
SELECT substr(ascs.fullkey, 3) AS asc_path
FROM json_tree('{"tcl":{"compat":""}}') AS ascs
WHERE "key" IS NOT NULL;
~~~

and its output

| asc_path   |
|------------|
| tcl        |
| tcl.compat |

This query takes the tcl/compat prefix as a specially encoded JSON object, returns one column from the *json_tree* output, and strips the leading '$.' prefix. The result contains ancestor paths (for example, for category path 'tcl/compat/zlib1/') with a period used as the path separator, which can be straightforwardly converted to the desired result.

The special JSON object format

~~~json
{"tcl":{"compat":""}}
~~~

follows the "nested doll" design, with each node name acting as a JSON object key. The depth of the ancestor node in the path is mapped to the JSON object nesting level. The deepest level value is irrelevant and is set to the blank string. Note that even though the structure is nested, it can be constructed from the path by replacing the path separator with '":{"' and adding opening and closing pieces. At the end, there is a string of closing curly braces, and its length matches the number of levels. Here is the full query:

~~~sql
WITH
    json_nodes(ids) AS (
        VALUES
            ('["tcl/compat/zlib1/", "tcl/pkgs/thread2.8.7/tcl/cmdsrv/"]')
    ), 
    nodes AS (
        SELECT node_ids.value AS node_id
        FROM json_nodes AS jn, json_each(jn.ids) AS node_ids
    ),
    tops AS (SELECT node_id AS path FROM nodes),
    prefixes AS (
        SELECT ascii_id, prefix,
            length(prefix) - length(replace(prefix, '/', '')) AS depth
        FROM categories WHERE path IN tops
    ),    
    json_prefixes AS (
        SELECT *, json('{"' || replace(rtrim(prefix, '/'), '/', '": {"') ||
            '":""' || replace(hex(zeroblob(depth)), '00', '}')) AS prefix_json
        FROM prefixes
    ),
    ancestors AS (
        SELECT ascii_id,
            replace(replace(substr(fullkey, 3), '.', '/'), '^#^', '.') || '/' AS asc_path
        FROM
            json_prefixes AS jp,
            json_tree(replace(jp.prefix_json, '.', '^#^')) AS prefixes
        WHERE prefixes.parent IS NOT NULL
    )
SELECT * FROM ancestors;
~~~

Blocks *json_nodes* through *tops* constitute the same prologue as [before](select-desc#prologue). The *prefixes* block retrieves node prefixes from the *categories* table (alternatively, the query could incorporate the [path splitting template](../patterns/split-dsv#Split-Path)). *json_prefixes* prepares "nested doll" JSON objects. The *ancestors* block generates ancestor lists, temporarily replacing any periods in names with '^#^'  at the beginning and reversing it at the end. Note that if the input contains multiple paths with a shared prefix, the *asc_path* will have duplicate values.

**Output**

| ascii_id | asc_path                  |
|----------|---------------------------|
| 0FDAF2C8 | tcl/                      |
| 0FDAF2C8 | tcl/compat/               |
| BE0A8514 | tcl/                      |
| BE0A8514 | tcl/pkgs/                 |
| BE0A8514 | tcl/pkgs/thread2.8.7/     |
| BE0A8514 | tcl/pkgs/thread2.8.7/tcl/ |

---

A similar query splits provided paths directly:

~~~sql
WITH
    json_nodes(ids) AS (
        VALUES
            ('["tcl/compat/zlib1/", "tcl/pkgs/thread2.8.7/tcl/cmdsrv/"]')
    ), 
    nodes AS (
        SELECT "key" + 1 AS id, value AS node_id
        FROM json_nodes AS jn, json_each(jn.ids) AS node_ids
    ),
    tops AS (SELECT id, node_id AS path FROM nodes),
    levels AS (
        SELECT id, path, length(path) - length(replace(path, '/', '')) AS depth
        FROM tops
    ),
    json_objs AS (
        SELECT *, json('{"' || replace(rtrim(path, '/'), '/', '": {"') ||
            '":""' || replace(hex(zeroblob(depth)), '00', '}')) AS json_obj
        FROM levels
    ),
    ancestors AS (
        SELECT min(jo.id) AS id,
            replace(replace(substr(fullkey, 3), '.', '/'), '^#^', '.') || '/' AS asc_path
        FROM
            json_objs AS jo,
            json_tree(replace(jo.json_obj, '.', '^#^')) AS terms
        WHERE terms.parent IS NOT NULL
		GROUP BY asc_path
		ORDER BY id, asc_path
    )
SELECT * FROM ancestors;
~~~

This query splits paths as provided and no longer accesses the *categories* table. Additionally, the *nodes* block labels paths with array element id (instead of *ascii_id*) generated by *json_each*, and the *ancestors* block removes duplicate terms.

**Output**

| id | asc_path                         |
|----|----------------------------------|
| 1  | tcl/                             |
| 1  | tcl/compat/                      |
| 1  | tcl/compat/zlib1/                |
| 2  | tcl/pkgs/                        |
| 2  | tcl/pkgs/thread2.8.7/            |
| 2  | tcl/pkgs/thread2.8.7/tcl/        |
| 2  | tcl/pkgs/thread2.8.7/tcl/cmdsrv/ |

