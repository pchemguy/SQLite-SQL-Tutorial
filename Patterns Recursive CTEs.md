---
layout: default
title: Recursive CTEs
nav_order: 8
parent: Design Patterns
permalink: /patterns/rec-cte
---

By design, SQL delegates statement-level flow control to the database engine, so the standard SQL grammar does not include any flow control structures. The only exception is expression-level branching control. This control structure has operator and function representations, so it naturally integrates within an expression (similar to other functions/operators). Adding statement-level flow controls, such as branching or loops, necessitates grammar extension or special conventions. Recursive CTEs (RCTEs) follow the latter approach and implement the while/repeat loop structure without special grammar/syntax. The result is quite contrived; thus, comprehending and mastering RCTEs may not be straightforward.

A general loop structure has two essential code blocks - initialization and loop body. Conventionally, the RCTE analogous blocks must fit within a single WITH clause member block. Because the latter only supports SELECT-type members, an RCTE must also represent a valid SELECT statement. At the same time, RCTEs, like SQL in general, focus on row sets/views manipulation. The idea is that the initialization query generates a starting view used as one of the sources by the first loop cycle. Each execution of the loop body yields a new view used as one of the sources by the following loop cycle. Accordingly, both code blocks must represent complete SELECT statements. And because the two SELECT blocks must form a valid SELECT query, they should be glued together via a set operator, such as UNION.

By convention, the resulting view of an RCTE query is the compound consisting of the row set produced by the initialization query combined with all row sets returned by individual loop cycles using the specified set operator (the operator separating the initialization and the body blocks). This convention permits, for example, retrieving a subtree from a hierarchy modeled with the parent reference pattern.

Yet another convention is related to the meaning of the RCTE name. Outside the RCTE, its name refers to the resulting view above. This meaning is the same for both ordinary and recursive CTEs. An ordinary CTE cannot use its name as one of its sources. An RCTE, on the other hand, must use its name. Within an RCTE, its name refers to the processing frame/view, which, to a first approximation, contains the row set returned by the previous cycle or the initialization query for the first cycle. In other words, the inner and outer references to the RCTE name mean two completely different things.

Another essential feature of a loop structure is loop body start/end markers. By convention, the RCTE loop body starts with the first SELECT query portion that uses the RCTE name as one of its sources. The loop body must incorporate all code through the end of the RCTE code block, with all preceding code forming the initialization query. The body part may be a compound SELECT query combined using the same set operator as the one that combines the body and the initialization code.

There are two possible termination conditions. If the loop body includes the LIMIT clause, the loop terminates after the specified number of rows is processed. Alternatively, the loop terminates after processing all previously returned rows, with the loop body returning empty row sets.

<!--
 More precisely, the processing frame is a single row view (in a sense, a pointer to the next exiting row of the processing queue) sitting at the tip of a dynamically generated processing queue, as discussed later.

[Recursive CTEs][] is
-->

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
