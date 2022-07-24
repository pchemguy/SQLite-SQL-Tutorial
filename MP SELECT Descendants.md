---
layout: default
title: SELECT Descendants
nav_order: 3
parent: Materialized Paths
permalink: /mat-paths/select-desc
---

### Scope of provided selection queries for the MPs model

This and the following sections provide a family of standard MPs queries for selecting a set of category tree nodes, their descendants, and ancestors. These queries return rows from the *categories* table for the following standard MPs node sets:

  - root nodes (just the nodes from the input)
  - all descendants, excluding the root nodes
  - subtrees, including root nodes and descendants
  - children, i.e., direct descendants only

All provided selection queries have the [modular CTEs-based](../meta/db-derived-cte) structure and take a unified input in the form of a [JSON-array-formatted](../patterns/json-sql-input) string

~~~json
["tcl/compat/zlib1/", "BAZ/dev/git4win/x32/mingw32/share/"]
~~~

with a list of category *path* values, identifying subtree root nodes.

### Query structure and alternative category identifiers

Each query starts with a prologue. Its function is to take the unified JSON array string of category identifiers (the *json_nodes* entry block) and its exit *tops* block produces a table containing the *path* column with one category path per row:

#### Prologue input

~~~json
["tcl/compat/zlib1/", "BAZ/dev/git4win/x32/mingw32/share/"]
~~~

#### Prologue

~~~sql
WITH
    json_nodes(ids) AS (
        VALUES
            ('["tcl/compat/zlib1/", "BAZ/dev/git4win/x32/mingw32/share/"]')
    ), 
    nodes AS (
        SELECT node_ids.value AS node_id
        FROM json_nodes AS jn, json_each(jn.ids) AS node_ids
    ),
    tops AS (SELECT node_id AS path FROM nodes)
SELECT * FROM tops ORDER BY path;
~~~

#### Prologue output

| path                               |
|------------------------------------|
| BAZ/dev/git4win/x32/mingw32/share/ |
| tcl/compat/zlib1/                  |

---

Query prologue decouples the input format details from the query business logic. With query prologue and its output format defined, the input format and query business logic can be modified indepently. For example, switching from path-based to ascii_id-based identifies requires a localized change to a single block *tops*, identical for all queries in this family (immediate input in *json_nodes* is also adjusted; in production, a query parameter replaces this string input):

#### Input

~~~json
["0FDAF2C8","65887f45"]
~~~

#### Prologue

~~~sql
WITH
    json_nodes(ids) AS (
        VALUES
            ('["0FDAF2C8", "65887f45"]')
    ), 
    nodes AS (
        SELECT node_ids.value AS node_id
        FROM json_nodes AS jn, json_each(jn.ids) AS node_ids
    ),
    tops AS (
        SELECT cats.path
        FROM categories AS cats, nodes
        WHERE cats.ascii_id IN (nodes.node_id)
    )
SELECT * FROM tops ORDER BY path;
~~~

---

### MPs queries

Note that the four queries have alomst identical code, with the only difference in the WHERE clause (*records*). Here is a query template:

~~~sql
WITH
    json_nodes(ids) AS (
        VALUES
            ('["tcl/compat/zlib1/", "BAZ/dev/git4win/x32/mingw32/share/"]')
    ), 
    nodes AS (
        SELECT node_ids.value AS node_id
        FROM json_nodes AS jn, json_each(jn.ids) AS node_ids
    ),
    tops AS (SELECT node_id AS path FROM nodes),
    records AS (
        SELECT cats.*
        FROM categories AS cats, tops
        WHERE %SELECTOR%
    )
SELECT * FROM records ORDER BY path;
~~~

in which %SELECTOR% should be replaced according to the following table

<pre>
| node set    | %SELECTOR%                          |
|-------------|-------------------------------------|
| root nodes  | `cats.path    IN  (tops.path)`      |
| descendants | `cats.prefix like tops.path || '%'` |
| subtrees    | `cats.path   like tops.path || '%'` |
| children    | `cats.prefix  =   tops.path`        |
</pre>
