---
layout: default
title: Recursive CTEs
nav_order: 8
parent: Design Patterns
permalink: /patterns/rec-cte
---

[Recursive CTEs][] is

~~~sql
WITH RECURSIVE
	folders(path_old) AS (
		VALUES
			('doc/thesis/exp'),
			('doc/thesis/theory'),
			('doc/app/job/lor'),
			('code/scripts/py'),
			('code/scripts/bas')
	),
    ops(opid, rootpath_old, rootpath_new) AS (
        VALUES
			(1, 'doc/',            'docABC'                 ),
			(2, 'docABC/thesis/',  'docABC/master'          ),
			(3, 'docABC/app/job/', 'docABC/app/academic_job'),
			(4, 'code/',           'prog'                   )
    ),
    LOOP_COPY AS (
            SELECT 0 AS opid, path_old AS path_new
            FROM folders
        UNION ALL
            SELECT ops.opid, path_new
            FROM LOOP_COPY AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid,
				   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    
    -------------------------------------------------------------------------------
    -------------------------------------------------------------------------------
    LOOP_COPY_INIT AS (
			SELECT 0 AS opid, path_old AS path_new
			FROM folders
    ),
    LOOP_COPY_STEP_1 AS (
            SELECT ops.opid, path_new
            FROM LOOP_COPY_INIT AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid,
				   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY_INIT AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_2 AS (
            SELECT ops.opid, path_new
            FROM LOOP_COPY_STEP_1 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid,
				   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY_STEP_1 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_3 AS (
            SELECT ops.opid, path_new
            FROM LOOP_COPY_STEP_2 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid,
				   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY_STEP_2 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_4 AS (
            SELECT ops.opid, path_new
            FROM LOOP_COPY_STEP_3 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid,
				   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY_STEP_3 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    LOOP_COPY_STEP_5_STOP AS (
            SELECT ops.opid, path_new
            FROM LOOP_COPY_STEP_4 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
        UNION ALL
            SELECT ops.opid,
				   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY_STEP_4 AS BUFFER, ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    -------------------------------------------------------------------------------
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
-- SELECT * FROM LOOP_COPY_STEP_4;
-- SELECT * FROM LOOP_COPY_STEP_5_STOP;
SELECT * FROM LOOP_COPY;
~~~


<!-- References -->

[Recursive CTEs]: https://sqlite.org/lang_with.html#recursive_common_table_expressions
