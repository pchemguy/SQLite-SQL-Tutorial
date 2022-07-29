---
layout: default
title: Recursive CTEs
nav_order: 8
parent: Design Patterns
permalink: /patterns/rec-cte
---

~~~sql
WITH RECURSIVE
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
            trim(json_extract(value, '$.path_new'), '/') || '/' AS rootpath_new
        FROM json_ops AS jo, json_each(jo.ops) AS terms
    ),
    subtrees_old AS (
        SELECT opid, ascii_id, path AS path_old
        FROM base_ops, categories
        WHERE path like rootpath_old || '%'
        ORDER BY opid, path
    ),
    LOOP_COPY AS (
            SELECT 0 AS opid, ascii_id, path_old AS path_new
            FROM subtrees_old
        UNION ALL
            SELECT ops.opid, ascii_id, path_new
            FROM LOOP_COPY AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid, '~' || ascii_id AS ascii_id,
                   replace(path_new, rootpath_old, rootpath_new) AS path_new
            FROM LOOP_COPY AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    
    -------------------------------------------------------------------------------
    LOOP_COPY_INIT AS (
        SELECT 0 AS opid, ascii_id, path_old AS path_new
        FROM subtrees_old
    ),
    LOOP_COPY_STEP_1 AS (
            SELECT ops.opid, ascii_id, path_new
            FROM LOOP_COPY_INIT AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid, '~' || ascii_id AS ascii_id,
                   replace(path_new, rootpath_old, rootpath_new) AS path_new
            FROM LOOP_COPY_INIT AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_2 AS (
            SELECT ops.opid, ascii_id, path_new
            FROM LOOP_COPY_STEP_1 AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid, '~' || ascii_id AS ascii_id,
                   replace(path_new, rootpath_old, rootpath_new) AS path_new
            FROM LOOP_COPY_STEP_1 AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_3 AS (
            SELECT ops.opid, ascii_id, path_new
            FROM LOOP_COPY_STEP_2 AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid, '~' || ascii_id AS ascii_id,
                   replace(path_new, rootpath_old, rootpath_new) AS path_new
            FROM LOOP_COPY_STEP_2 AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_STOP AS (
            SELECT ops.opid, ascii_id, path_new
            FROM LOOP_COPY_STEP_3 AS BUFFER, base_ops AS ops
            WHERE ops.opid = (SELECT max(base_ops.opid) + 1 FROM base_ops)
        UNION ALL
            SELECT ops.opid, '~' || ascii_id AS ascii_id,
                   replace(path_new, rootpath_old, rootpath_new) AS path_new
            FROM LOOP_COPY_STEP_3 AS BUFFER, base_ops AS ops
            WHERE ops.opid = (SELECT max(base_ops.opid) + 1 FROM base_ops)
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    -------------------------------------------------------------------------------
    
    subtrees_new_base AS (
        SELECT ascii_id, path_new
        FROM LOOP_COPY
        WHERE opid = (SELECT max(opid) FROM base_ops)
          AND length(ascii_id) > 8
        GROUP BY ascii_id, path_new
        ORDER BY path_new
    )
    
-- SELECT * FROM subtrees_new_base;
-- SELECT * FROM LOOP_COPY_INIT;
-- SELECT * FROM LOOP_COPY_STEP_1;
-- SELECT * FROM LOOP_COPY_STEP_2;
-- SELECT * FROM LOOP_COPY_STEP_3;
SELECT * FROM LOOP_COPY_STEP_STOP;
~~~
