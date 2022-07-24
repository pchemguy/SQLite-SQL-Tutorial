---
layout: default
title: SELECT Descendants
nav_order: 3
parent: Materialized Paths
permalink: /mat-paths/select-desc
---

### Scope of provided selection queries for the MPs model

Current and the following sections provide a family of standard MPs queries for selecting a set of category tree nodes, their descendants, and ancestors. These queries return rows from the *categories* table for the following standard MPs node sets:

  - root nodes (just the nodes from the input)
  - subtrees, including root nodes and descendants
  - all descendants, excluding the root nodes
  - children, i.e., direct descendants only

All provided selection queries have the [modular CTEs-based](../meta/db-derived-cte) structure and take a unified input in the form of a [JSON-array-formatted](../patterns/json-sql-input) string

~~~json
["tcl/compat/zlib1/", "BAZ/dev/git4win/x32/mingw32/share/"]
~~~

with a list of category *path* values, identifying subtree root nodes.

### Query structure and alternative category identifiers

Each query starts with a prologue. Its function is to take a unified JSON array string with category identifiers (the *json_nodes* entry block), and its exit *tops* block produces a table containing the *path* column with one category path per row:

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

The query prologue decouples the input format details from the query business logic, permitting their independent modification. For example, switching from path-based to ascii_id-based identifies requires a localized change to a single block *tops*, identical for all queries in this family (immediate input in *json_nodes* is also adjusted; in production, a query parameter replaces this string input):

#### Input - *ascii_id*

~~~json
["0FDAF2C8","65887f45"]
~~~

#### Prologue - *ascii_id*

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
    tops AS (SELECT path FROM categories WHERE ascii_id IN nodes)
SELECT * FROM tops ORDER BY path;
~~~

---

### Selection queries

The following query selects just the nodes from the input (root nodes).

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
    records AS (SELECT * FROM categories WHERE path IN nodes)
SELECT * FROM records ORDER BY path;
~~~

The remaining three queries in this section have almost identical code, with the only difference in the WHERE clause (*records*). In the query template below, replace %SELECTOR% according to the following table:

<pre>
|  <b>node set</b>   |             <b>%SELECTOR%</b>            |
|-------------|-----------------------------------|
| subtrees    | cats.path   like tops.path || '%' |
| descendants | cats.prefix like tops.path || '%' |
| children    | cats.prefix  =   tops.path        |
</pre>

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
