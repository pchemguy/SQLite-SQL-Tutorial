---
layout: default
title: DELETE Subtrees
nav_order: 7
parent: Materialized Paths
permalink: /mat-paths/delete
---

The following query performs a simple deletion via the database DELETE statement and associated cascades. This query contains a [modify-style prologue](modify#prologue), and the *targets* block prepares a list of deleted record IDs (the DELETE statement, as opposed to UPDATE, does not support joins).

~~~sql
WITH
    json_ops(ops) AS (
        VALUES
            (json(
                '['                                                                    ||
                    '{"op":"delete", "path_old":"tcl/compat/zlib1/"},'                 ||
                    '{"op":"delete", "path_old":"BAZ/dev/git4win/x32/mingw32/share/"}' ||
                ']'
            ))
    ),
    base_ops AS (
        SELECT
            "key" + 1 AS opid,
            json_extract(value, '$.op') AS op,
            json_extract(value, '$.path_old') AS path_old,
            json_extract(value, '$.path_new') AS path_new
        FROM json_ops AS jo, json_each(jo.ops) AS terms
    ),
    targets AS (SELECT path FROM categories, base_ops WHERE path like path_old || '%')
DELETE FROM categories WHERE path IN targets;
~~~
