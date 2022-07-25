---
layout: default
title: Delimiter-Separated Values
nav_order: 2
parent: Design Patterns
permalink: /patterns/split-dsv
---

Note that the scripts in this section rely on the SQLite JSON library. In the latest version of SQLite, the JSON library is part of the core and is included in the build by default. Until recently, however, JSON has been developed as a loadable extension, which was not part of the default builds. Verify that the JSON functionality is available before using/adapting code from this tutorial (for example, use these [queries](/meta/engine)).

### Split delimiter-separated value strings via JSON

SQLite does not have a string splitting function, so, for example, splitting a file system path ('/usr/share/info' or 'C:/Winows/System32/drivers/etc/') into directories is not straightforward. The JSON library helps compensate for this missing functionality.

The basic procedure is as follows:

 1. Transform the delimited string to the JSON array format (preprocessing):
'C:/Winows/System32/drivers/etc/' &rarr; '["C:", "Winows", "System32", "drivers", "etc"]'
    - *trim* strips leading and trailing separators;
    - *replace* changes the original separator to '", "' (with or without the space);
    - *concat* adds '["' and '"]'.
 2. Process JSON-array-formatted string
     - *json_each* table-valued function splits the preprocessed string; the relevant columns include
	     - *key*, containing a zero-based term index, going from left to right, and
	     - *value*, containing individual components.

This query

<a name="DSV-Query"></a>
~~~sql
SELECT "terms"."key" AS term_id, "terms"."value" AS term
FROM json_each('["' || replace(trim('C:/Winows/System32/drivers/etc/', '/'), '/', '", "') || '"]') AS terms;
~~~

produces output

| term_id | term     |
|---------|----------|
| 0       | C:       |
| 1       | Winows   |
| 2       | System32 |
| 3       | drivers  |
| 4       | etc      |

### Split path into prefix and name

The following query splits the *path* column in the source table into *prefix* (parent path) and *name*. The first *paths* block defines mock input, and the two other blocks perform actual splitting.

<a name="Split-Path"></a>
~~~sql
WITH
    paths(path) AS (VALUES ('tcl/compat/zlib1/'), ('tcl/pkgs/thread2.8.7/tcl/cmdsrv/')), 
    node_names AS (
        SELECT path,
            json_extract('["' || replace(trim(path, '/'), '/', '", "') || '"]', '$[#-1]') AS name
        FROM paths
    ),
    nodes AS (
        SELECT path, substr(path, 1, length(path) - length(name) - 1) AS prefix, name
        FROM node_names
	)
SELECT * FROM nodes;
~~~

**Output**

| path                             | prefix                    | name   |
|----------------------------------|---------------------------|--------|
| tcl/compat/zlib1/                | tcl/compat/               | zlib1  |
| tcl/pkgs/thread2.8.7/tcl/cmdsrv/ | tcl/pkgs/thread2.8.7/tcl/ | cmdsrv |
