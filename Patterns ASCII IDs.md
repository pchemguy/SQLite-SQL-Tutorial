---
layout: default
title: ASCII IDs
nav_order: 7
parent: Design Patterns
permalink: /patterns/ascii-id
---

The following query generates random eight-character-long ASCII IDs (*id_counts*.*id_counter* pieces) using the specified alphabet (*char_templates*). The dash and underscore characters extend the alphanumeric alphabet to 64 characters allowing the selection of the six least significant bits from a random 64-bit number via bitwise AND (instead of *mod*). The *ids* block generates a 64-bit integer representation of the same ID. Saving character code before grouping in *ascii_ids* would not work because the query optimizer tends to rerun the query, executing the *random* function and generating the character and character code independently. The intermediate table aggregation operation prevents this optimization.

<a name="ascii_id_gen"></a>
~~~sql
WITH
    id_counts(id_counter) AS (VALUES (3)),
    json_templates AS (SELECT '[' || replace(hex(zeroblob(id_counter*8-1)), '00', '0,') || '0]' AS json_template FROM id_counts),
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
    )
SELECT * FROM ids;
~~~

**Sample output**

| counter | ascii_id | bin_id              |
|---------|----------|---------------------|
| 0       | qszS_Wed | 8175012247544227167 |
| 1       | HwrjLnvG | 5221768093833786951 |
| 2       | bXHJexP0 | 7086493498034638896 |
