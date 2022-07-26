---
layout: default
title: CREATE Paths
nav_order: 6
parent: Materialized Paths
permalink: /mat-paths/create
---

The CREATE script incorporates the following components:

  - [prologue](modify#prologue) - __*json_ops* through *base_ops*__;
  - [ancestor list](select-asc#list-ancestors) generation - __*levels* through *ancestors*__ (note that the referenced template contains SELECT-style prologue, which is replaced with the modify-style prologue);
  - *path_terms* removes rows corresponding to already existent categories and adds row numbers;
  - [ASCII id](../patterns/ascii-id) generation - __*id_counts* through *ids*__;
  - *new_nodes* joins path terms with newly generated IDs.

~~~sql
WITH
    json_ops(ops) AS (
        VALUES
            (json(
                '['                                                            ||
                    '{"op":"create", "path_new":"BAZ/bld/tcl/tests/safe/"},'   ||
                    '{"op":"create", "path_new":"safe/ssub00/modules/"},'      ||
                    '{"op":"create", "path_new":"safe/ssub00/msys/nix/misc/"}' ||
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
    levels AS (
        SELECT opid, path_new AS path, length(path_new) - length(replace(path_new, '/', '')) AS depth
        FROM base_ops
    ),
    json_objs AS (
        SELECT *, json('{"' || replace(rtrim(path, '/'), '/', '": {"') ||
            '":""' || replace(hex(zeroblob(depth)), '00', '}')) AS json_obj
        FROM levels
    ),
    ancestors AS (
        SELECT min(jo.opid) AS opid,
            replace(replace(substr(fullkey, 3), '.', '/'), '^#^', '.') || '/' AS asc_path,
            replace("key", '^#^', '.') AS asc_name
        FROM
            json_objs AS jo,
            json_tree(replace(jo.json_obj, '.', '^#^')) AS terms
        WHERE terms.parent IS NOT NULL
        GROUP BY asc_path
        ORDER BY opid, asc_path
    ),
    path_terms AS (
        SELECT 
            row_number() OVER (ORDER BY opid, asc_path) AS counter,
            ancestors.*, substr(asc_path, 1, length(asc_path) - length(asc_name) - 1) AS asc_prefix
        FROM ancestors
        LEFT JOIN categories AS cats ON asc_path = cats.path
        WHERE cats.ascii_id IS NULL        
    ),
    id_counts(id_counter) AS (SELECT count(*) FROM path_terms),
    json_templates AS (SELECT '[' || replace(hex(zeroblob(id_counter*8/2-1)), '0', '0,') || '0,0]' AS json_template FROM id_counts),
    char_templates(char_template) AS (VALUES ('-0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz_')),
    ascii_ids AS (
        SELECT group_concat(substr(char_template, (random() & 63) + 1, 1), '') AS ascii_id, "key"/8 + 1 AS counter
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
    new_nodes AS (
        SELECT bin_id AS id, asc_name AS name, asc_prefix AS prefix
		FROM path_terms, ids USING (counter)
	)
INSERT INTO categories (id, name, prefix)
SELECT * FROM new_nodes;
~~~