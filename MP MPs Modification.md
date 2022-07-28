---
layout: default
title: MPs Modification
nav_order: 5
parent: Materialized Paths
permalink: /mat-paths/modify
---

The standard modification operations include

  - create category path
  - delete category subtree
  - copy category subtree
  - move category subtree

### Unified JSON input format

The unified JSON input string supports the specification of multiple successive operations using a [JSON array of objects](../patterns/json-sql-input#json-array-object):

~~~json
[
  {"op":"create|delete|copy|move", "path_old":"%old_path#A%", "path_new":"%new_path#A%"},
  ...
  {"op":"create|delete|copy|move", "path_old":"%old_path#Z%", "path_new":"%new_path#Z%"}
]
~~~

### Prologue

~~~sql
WITH
    json_ops(ops) AS (
        VALUES
            (json(
                '['                                                                                 ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/safe00/", "path_new":"safe00/"},'  ||
                    '{"op":"move", "path_old":"safe00/",                   "path_new":"safe/"},'    ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/safe11/", "path_new":"safe11/"}'   ||
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
    )
SELECT * FROM base_ops;
~~~

**Output**

| opid | op   | path_old                  | path_new |
|------|------|---------------------------|----------|
| 1    | move | BAZ/bld/tcl/tests/safe00/ | safe00/  |
| 2    | move | safe00/                   | safe/    |
| 3    | move | BAZ/bld/tcl/tests/safe11/ | safe11/  |

---

### Conventions

**CREATE/INSERT** operation specifies *path_new* for each new category. The handling script must create any nonexistent ancestors (intermediate categories) automatically.

**DELETE** operation specifies *path_old* for each deleted category subtree. The database enigne also deletes all category assignment records in many-to-many (MtoM) tables for each deleted category via the foreign key cascades.

**COPY/MOVE/UPDATE** operations specify *path_old* and *path_new* for each affected category (similarly to FS copy/move operations on  directories). Special consideration is necessary for handling collisions. 

I follow the "merge on collide" convention. When the name of a moved/copied/renamed category collides with an existing one, preferably keep the latter. As long as the only category identification used outside the database is its name-based path, any external application would see any difference no matter which conflicting copy remains. So long as the relationship tables use the category's name-based path as the foreign key reference, all records in the MtoM tables can be updated following a simple protocol as if no collision has occurred. Explicit updating of the MtoM tables is still necessary because cascades will not work for merged nodes.

The remaining question is whether the COPY script should copy the associated MtoM records. On the one hand, FS routines usually copy contained files by default. On the other hand, the FS analogy has its limitations, as discussed [previously](../mat-paths/design-rules). Categories only act as containers/directories for other categories, in which case the associated FS logic is relevant to the behavior of the category system. The relationship between categories and categorized items is naturally different from the relationship between directories and files. For this reason, both approaches (copying the associated relationship records following the FS logic and not doing so) may have supporting arguments. The current implementation of the COPY operation does not copy the associated MtoM records.

### Foreing Keys

A common approach to executing a multi-statement modification operation that might temporarily violate foreign key constraints (FKCs) is via the [foreign_keys pragma][]. The [defer_foreign_keys pragma][] provides a safer option. This pragma enables a temporary violation of FKCs within an active transaction. However, the database prevents committing a transaction violating any FKCs, and resets the *defer_foreign_keys* pragma automatically at each COMMIT/ROLLBACK, leaving foreign keys enabled at transaction conclusion.


<!-- References -->
[foreign_keys pragma]: https://sqlite.org/pragma.html#pragma_foreign_keys
[defer_foreign_keys pragma]: https://sqlite.org/pragma.html#pragma_defer_foreign_keys 
