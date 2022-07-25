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

**DELETE** operation specifies *path_old* for each deleted category subtree. In addition to deleting the specified categories and all their descendants, the script must also delete all category assignment records for each deleted category in many-to-many (MtoM) tables. Script's business logic depends on the specific delete operation scenario.

A simple approach involves deletion of all associated category rows from the *categories*  table. Because each many-to-many table should include a foreign key to the *categories* table with cascaded deletions, all category assignment operations will be deleted automatically by the database.

An advanced approach implements the delete operation as *move-to-trash*. In this case, the script uses the MOVE operation, relocating the target subtrees into the system *~Trash* category and using the database UPDATE statement instead of DELETE. The script also appends '~' || *ascii_id* to the names of deleted subtree root nodes to avoid potential name collisions. Here, the deletion of the associated category assignment records from MtoM tables necessitates explicit processing because the UPDATE statement does not trigger deletion cascades.

Recall that SQLite does not support dynamic SQL, in general, or conversion of a string name into an object identifier, in particular. For this reason, a library-style script for advanced deletion necessitates tailoring it to a specific set of MtoM tables. Strictly speaking, there is an alternative database protocol (DELETE/INSERT) for implementing the *move-to-trash* operation, which triggers the deletion cascades and, thus, permits the creation of a universal script. Additional planned extensions/features in the future may not be fully compatible with cascaded operations. For this reason, the developed scripts handle relationship tables explicitly, and their simplified versions (not given) let cascades do their part.

**COPY/MOVE/UPDATE** operations specify *path_old* and *path_new* for each affected category (similarly to FS copy/move operations on  directories). Special consideration is necessary for handling collisions. I follow the "merge on collide" convention. When the name of a moved/copied/renamed category collides with an existing one, preferably keep existing copy. As long as the only category identification used outside the database is its name-based path, any external application would not be able to see any difference either way. So long as the relationship table uses the category's name-based path as the foreign key reference, all records in the MtoM table can be updated following a simple protocol as if no collision has occurred. Explicit updating of the MtoM tables is still necessary because cascaded updates would fail for merged nodes.

The remaining question is whether the COPY script should copy the associated MtoM records. On the one hand, FS routines usually copy contained files by default. On the other hand, the FS analogy has its limitations, as discussed [previously](../mat-paths/design-rules). Categories only act as containers/directories for other categories, in which case the associated FS logic is relevant to the behavior of the category system. The relationship between categories and categorized items is naturally different from the relationship between directories and files. For this reason, both approaches (copying the associated relationship records following the FS logic and not doing so) may have supporting arguments. The current implementation of the COPY operation does not copy the associated MtoM records.

