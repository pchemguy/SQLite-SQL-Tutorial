---
layout: default
title: MOVE Subtrees
nav_order: 8
parent: Materialized Paths
permalink: /mat-paths/move
---

The MOVE script moves/renames specified categories. The script also needs to handle associated items when necessary. For non-existing new paths, the database updates item associations automatically via foreign key cascades. However, when a collision occurs (existing new path), the scripts keeps one copy only (preferably the existing one), deleting the rest, and must update item associations explicitly before deletion. The current script updates the *files_categories* demo table and needs to be adjusted for actual association tables.

The core transformation step uses an RCTE loop discussed in the [RCTEs for Recursive Modify](../patterns/rcte-modify) section. The MOVE operation does not create new paths as opposed to the COPY operation, so the loop code is simplified. Because the total number of existing category paths may be substantially higher than the number of affected paths, it is desirable to isolate the subset of affected paths. The *subtrees_old* CTE achieves this goal by matching the category path prefix against the *path_old* variable (*ops*).

Postprocessing code includes two filters. The *subtrees_path* CTE matches the old and new paths and filters out unchanged categories. The *new_paths* CTE labels category rows for which path already exists in the *categories* table or duplicate *path_new* entries are present in the *subtrees_new_base*. The final set is used to update the *categories* table and associated relationship tables.

~~~sql
CREATE TEMP TABLE IF NOT EXISTS move_targets (
    ascii_id, path_old, path_new, prefix_new, name_new, target_exists
);
DELETE FROM temp.move_targets;

WITH RECURSIVE
    ------------------------------ PROLOGUE ------------------------------
    json_ops(ops) AS (
        VALUES
            (json(
                '['                                                                                                  ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/safe00/", "path_new":"safe00/"},'                   ||
                    '{"op":"move", "path_old":"safe00/",                   "path_new":"safe/"},'                     ||
                    '{"op":"move", "path_old":"BAZ/dev/msys2",             "path_new":"BAZ/dev/msys/"},'             ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/preEEE/", "path_new":"preEEE/"},'                   ||
                    '{"op":"move", "path_old":"safe/modules/",             "path_new":"safe/modu/"},'                ||
                    '{"op":"move", "path_old":"safe/modu/mod2/",           "path_new":"safe/modu/mod3/"},'           ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/ssub00/", "path_new":"safe/ssub00/"},'              ||
                    '{"op":"move", "path_old":"BAZ/dev/msys/mingw32/",     "path_new":"BAZ/dev/msys/nix/"},'         ||
                    '{"op":"move", "path_old":"safe/ssub00/modules/",      "path_new":"safe/modules/"},'             ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/manYYY/", "path_new":"man000/"},'                   ||
                    '{"op":"move", "path_old":"BAZ/dev/msys/nix/etc/",     "path_new":"BAZ/dev/msys/nix/misc/"},'    ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/manZZZ/", "path_new":"BAZ/bld/tcl/tests/man000/"},' ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/man000/", "path_new":"man000/"},'                   ||
                    '{"op":"move", "path_old":"BAZ/bld/tcl/tests/safe11/", "path_new":"safe11/"}'                    ||
                ']'
            ))
    ),
    base_ops AS (
        SELECT
            "key" + 1 AS opid,
            json_extract(value, '$.op') AS op,
            trim(json_extract(value, '$.path_old'), '/') || '/' AS rootpath_old,
            trim(json_extract(value, '$.path_new'), '/') AS rootpath_new
        FROM json_ops AS jo, json_each(jo.ops) AS terms
    ),
    /********************************************************************/
    --------------------------- SUBTREES LIST ----------------------------
    subtrees_old AS (
        SELECT opid, ascii_id, path AS path_old
        FROM base_ops, categories
        WHERE path_old like rootpath_old || '%'
        ORDER BY opid, path_old
    ),
    /********************************************************************/
    ----------------------------- MOVE LOOP ------------------------------
    LOOP_MOVE AS (
            SELECT 0 AS opid, ascii_id, path_old AS path_new
            FROM subtrees_old
        UNION ALL
            SELECT ops.opid, ascii_id,
                   iif(BUFFER.path_new NOT like rootpath_old || '%', path_new,
                       rootpath_new || substr(path_new, length(rootpath_old))
                   ) AS path_new
            FROM LOOP_MOVE AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
    ),
    /********************************************************************/
    subtrees_new_base AS (
        SELECT ascii_id, path_new,
               json_extract('["' || replace(trim(path_new, '/'), '/', '", "') || '"]', '$[#-1]') AS name_new
        FROM LOOP_MOVE
        WHERE opid = (SELECT max(base_ops.opid) FROM base_ops)
    ),
    subtrees_path AS (
        SELECT
            trnew.ascii_id, path_old, path_new,
            substr(path_new, 1, length(path_new) - length(name_new) - 1) AS prefix_new,
            name_new
        FROM subtrees_new_base AS trnew, subtrees_old AS trold
        WHERE trnew.ascii_id = trold.ascii_id
          AND path_old <> path_new
    ),
    new_paths AS (
        SELECT
            subtrees_path.*,
            (cats.ascii_id IS NOT NULL) + (row_number() OVER (PARTITION BY path_new) - 1) AS target_exists
        FROM subtrees_path LEFT JOIN categories AS cats ON path_new = path
    )
INSERT INTO temp.move_targets (ascii_id, path_old, path_new, prefix_new, name_new, target_exists)
SELECT * FROM new_paths ORDER BY target_exists, path_old;

PRAGMA defer_foreign_keys = 1;
SAVEPOINT "MOVE_CATS";

-- Collision-free category updates
UPDATE categories SET (name, prefix) = (name_new, prefix_new)
FROM temp.move_targets AS mvt
WHERE mvt.target_exists = 0
  AND mvt.path_old = categories.path;

-- Update association tables
UPDATE files_categories SET cat_id = path_new
FROM temp.move_targets AS mvt
WHERE mvt.target_exists = 1
  AND mvt.path_old = cat_id;

DELETE FROM categories
WHERE path IN (
    SELECT path_old
    FROM temp.move_targets AS mvt
    WHERE mvt.target_exists = 1
);

RELEASE "MOVE_CATS";
~~~
