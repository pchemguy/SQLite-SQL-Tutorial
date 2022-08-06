---
layout: default
title: COPY Subtrees
nav_order: 9
parent: Materialized Paths
permalink: /mat-paths/copy
---

The current convention for the COPY operation is to copy category subtrees only for each input category, but no category-item assignment. This convention simplifies the script

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
            trim(json_extract(value, '$.path_new'), '/') AS rootpath_new
        FROM json_ops AS jo, json_each(jo.ops) AS terms
    ),
    subtrees_old AS (
        SELECT opid, ascii_id, path AS path_old
        FROM base_ops, categories
        WHERE path_old like rootpath_old || '%'
        ORDER BY opid, path_old
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
                   rootpath_new || substr(path_new, length(rootpath_old)) AS path_new
            FROM LOOP_COPY AS BUFFER, base_ops AS ops
            WHERE ops.opid = BUFFER.opid + 1
              AND BUFFER.path_new like rootpath_old || '%'            
    ),
    subtrees_new_base AS (
        SELECT ascii_id, path_new
        FROM LOOP_COPY
        WHERE opid = (SELECT max(opid) FROM base_ops)
          AND length(ascii_id) > 8
        GROUP BY ascii_id, path_new
        ORDER BY path_new
    ),
    subtrees_path AS (
        SELECT path_new FROM subtrees_new_base GROUP BY path_new
    ),
    subtrees_new AS (
        SELECT
            json_extract('["' || replace(trim(path_new, '/'), '/', '", "') || '"]', '$[#-1]') AS name_new,
            path_new
        FROM subtrees_path LEFT JOIN categories AS cats ON path_new = path
        WHERE ascii_id IS NULL
    ),    
    new_paths AS (
        SELECT row_number() OVER (ORDER BY path_new) - 1 AS counter, 
               substr(path_new, 1, length(path_new) - length(name_new) - 1) AS prefix_new,
               subtrees_new.*
        FROM subtrees_new
    ),
    id_counts(id_counter) AS (SELECT count(*) FROM new_paths),
    json_templates AS (SELECT '[' || replace(hex(zeroblob(id_counter*8/2-1)), '0', '0,') || '0,0]' AS json_template FROM id_counts),
    char_templates(char_template) AS (VALUES ('-0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz_')),
    ascii_ids AS (
        SELECT group_concat(substr(char_template, (random() & 63) + 1, 1), '') AS ascii_id, "key"/8 AS counter
        FROM char_templates, json_templates, json_each(json_templates.json_template) AS terms
        GROUP BY counter
    ),
    ids AS (
        SELECT counter, ascii_id,
               (unicode(substr(ascii_id, 1, 1)) << 8*7) +
               (unicode(substr(ascii_id, 2, 1)) << 8*6) +
               (unicode(substr(ascii_id, 3, 1)) << 8*5) +
               (unicode(substr(ascii_id, 4, 1)) << 8*4) +
               (unicode(substr(ascii_id, 5, 1)) << 8*3) +
               (unicode(substr(ascii_id, 6, 1)) << 8*2) +
               (unicode(substr(ascii_id, 7, 1)) << 8*1) +
               (unicode(substr(ascii_id, 8, 1)) << 8*0) AS bin_id
        FROM ascii_ids
    ),
    new_nodes AS (SELECT bin_id AS id, name_new AS name, prefix_new AS prefix FROM new_paths, ids USING (counter))
INSERT INTO categories (id, name, prefix)
SELECT * FROM new_nodes;
~~~
