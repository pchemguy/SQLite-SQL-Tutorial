---
layout: default
title: Recursive CTEs
nav_order: 8
parent: Design Patterns
permalink: /patterns/rec-cte
---

Because, by design, SQL delegates the flow control of statement evaluation to the database engine, the standard SQL grammar does not include any flow control structures. The only exception is expression-level branching control. This control structure has operator and function representation, which is why it naturally integrates within an expression. Adding statement-level flow controls, such as branching or loops, necessitates grammar extension or special conventions. Recursive CTEs (RCTEs) follow the latter approach and implement the while/repeat loop structure. A general loop structure has four essential components: initialization code, the body of the loop, syntactic elements marking the beginning and end of the body code, and a means to exit the loop (termination conditions).
  
RCTEs do not introduce any special grammar/syntax; instead, it uses the standard SELECT statement syntax and a special convention. The result is quite contrived; for this reason, comprehending and mastering RCTEs may not be straightforward. Because the sole function of SQL is the construction and manipulation of row sets (views), the initialization and loop body blocks of an RCTE constitute complete SELECT statements. The initialization query generates starting view used as one of the sources by the body query. Each execution cycle of the loop body yields a new view used as one of the sources by the following loop cycle. There are two possible termination conditions. If the loop body includes the LIMIT clause, the loop terminates after the specified number of rows is processed. Alternatively, the loop terminates after processing all previously returned rows, with the loop body returning empty row sets.

By convention, initialization and loop body SELECT blocks should be part of the same CTE. Because a CTE is, in turn, a SELECT statement, a set operator (such as UNION) must glue the initialization and loop body blocks, yielding a compound SELECT. Naturally, the initialization block goes first, followed by the body. The body query, by convention, uses its CTE name as one of the sources in the FROM clause. The CTE may include multiple set operations, and the first SELECT block referencing its CTE's name constitutes marks the beginning of the body block, with all preceding code forming the initialization query.

The resulting view of an RCTE query is the compound consisting of the row set produced by the initialization query combined with all row sets returned by individual loop cycles using the specified set operator (the operator separating the initialization and the body blocks).

Yet another convention is related to the meaning of the RCTE name. Outside the RCTE, its name refers to the resulting view above. This meaning is the same for both ordinary and recursive CTEs. An ordinary CTE cannot use its name as one of its sources. Within an RCTE, on the other hand, its name refers to the processing frame/view, which, to a first approximation, contains the row set returned by the previous cycle or the initialization query for the first cycle. More precisely, the processing frame is a single row view (in a sense, a pointer to the next exiting row of the processing queue) sitting at the tip of a dynamically generated processing queue, as discussed later.

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
