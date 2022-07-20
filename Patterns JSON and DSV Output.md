---
layout: default
title: JSON and DSV Output
nav_order: 5
parent: Design Patterns
permalink: /patterns/json-sql-output
---

Using JSON, rather than a record set, for encoding the returned data may also be more efficient due to the reduced number of required API calls. While SQLite API is usually fast, each returned value still costs several API calls. (Under certain circumstances, however, there might be a high SQLite-independent overhead for each API call.) Another important consideration is whether there are any potential side effects of data conversion between numeric and textual formats. Let us show a few examples.

Consider a modified query from the [Surrogate Variables](/patterns/variables#DSV-Query) section:

~~~sql
WITH
    delimiters(delimiter) AS (VALUES ('/')),
    strings(string_id, string) AS (
        VALUES
            ('abc', 'C:/Winows/System32/drivers/etc/'),
            ('def', 'C:/Users/Public/Desktop')
    ),
    folders AS (
        SELECT string_id, "terms"."key" AS term_id, "terms"."value" AS term
        FROM
            delimiters, strings,
            json_each('["' || replace(trim(string, delimiter), delimiter, '", "') || '"]') AS terms
        ORDER BY string_id, term_id
    ),
    json_folders AS (
        SELECT string_id, json_group_array(term) AS path_json
        FROM folders
        GROUP BY string_id
    )
SELECT * FROM json_folders;
~~~

which outputs:

| string_id | path_json                                  |
|-----------|--------------------------------------------|
| abc       | ["C:","Winows","System32","drivers","etc"] |
| def       | ["C:","Users","Public","Desktop"]          |

This query has a new section, *json_folders*, at the end, which uses *json_group_array* to collect folders belonging to the same path. Note that the ordering clause is added to the *folders* section to ensure that the order of individual folders within JSON arrays after grouping reflects their positions in original paths.

---

Similarly, the following modified query:

~~~sql
WITH
    folders AS (
        SELECT
            json_extract(dirs.value, '$.bin_id') AS bin_id,
            json_extract(dirs.value, '$.prefix') AS prefix,
            json_extract(dirs.value, '$.name')   AS name
        FROM
            json_each(
                '['                                                                                    ||
                    '{"bin_id": "239", "prefix": "C:/Winows/System32/drivers/etc", "name": "hosts"},'  ||
                    '{"bin_id": "876", "prefix": "C:/Users/Public/Desktop",        "name": "pic"  },'  ||
                    '{"bin_id": "374", "prefix": "C:/Users/Default/Music",         "name": "drum" }'   ||
                ']'
            ) AS dirs
    ),
    json_fs_objects AS (
        SELECT
            json_group_array(json_object('bin_id', bin_id, 'prefix', prefix, 'name', name)) AS fs_objects
        FROM folders
	)
SELECT * FROM json_fs_objects;
~~~

return a scalar string containing a set of records in the JSON format:

~~~sql
[
    {"bin_id": "239", "name": "hosts", "prefix": "C:/Winows/System32/drivers/etc"},
    {"bin_id": "876", "name": "pic",   "prefix": "C:/Users/Public/Desktop"       },
    {"bin_id": "374", "name": "drum",  "prefix": "C:/Users/Default/Music"        }
]
~~~

JSON objects are a bit too verbous when returning a record set, but there are a few other options. For example:

~~~sql
WITH
    folders AS (
        SELECT
            json_extract(dirs.value, '$.bin_id') AS bin_id,
            json_extract(dirs.value, '$.prefix') AS prefix,
            json_extract(dirs.value, '$.name')   AS name
        FROM
            json_each(
                '['                                                                                    ||
                    '{"bin_id": "239", "prefix": "C:/Winows/System32/drivers/etc", "name": "hosts"},'  ||
                    '{"bin_id": "876", "prefix": "C:/Users/Public/Desktop",        "name": "pic"  },'  ||
                    '{"bin_id": "374", "prefix": "C:/Users/Default/Music",         "name": "drum" }'   ||
                ']'
            ) AS dirs
    ),
    tsv_fs_objects AS (
        SELECT group_concat(bin_id || x'09' || prefix || x'09' || name, x'0A') AS fs_objects
        FROM folders
    )
SELECT * FROM tsv_fs_objects;
~~~

or

~~~sql
WITH
    folders AS (
        SELECT
            json_extract(dirs.value, '$.bin_id') AS bin_id,
            json_extract(dirs.value, '$.prefix') AS prefix,
            json_extract(dirs.value, '$.name')   AS name
        FROM
            json_each(
                '['                                                                                    ||
                    '{"bin_id": "239", "prefix": "C:/Winows/System32/drivers/etc", "name": "hosts"},'  ||
                    '{"bin_id": "876", "prefix": "C:/Users/Public/Desktop",        "name": "pic"  },'  ||
                    '{"bin_id": "374", "prefix": "C:/Users/Default/Music",         "name": "drum" }'   ||
                ']'
            ) AS dirs
    ),
    tsv_fs_objects AS (
        SELECT
            group_concat(printf('%s' || x'09' || '%s' || x'09' || '%s', bin_id, prefix, name), x'0A') AS fs_objects
        FROM folders
    )
SELECT * FROM tsv_fs_objects;
~~~

or

~~~sql
WITH
    folders AS (
        SELECT
            json_extract(dirs.value, '$.bin_id') AS bin_id,
            json_extract(dirs.value, '$.prefix') AS prefix,
            json_extract(dirs.value, '$.name')   AS name
        FROM
            json_each(
                '['                                                                                    ||
                    '{"bin_id": "239", "prefix": "C:/Winows/System32/drivers/etc", "name": "hosts"},'  ||
                    '{"bin_id": "876", "prefix": "C:/Users/Public/Desktop",        "name": "pic"  },'  ||
                    '{"bin_id": "374", "prefix": "C:/Users/Default/Music",         "name": "drum" }'   ||
                ']'
            ) AS dirs
    ),
    records AS (
            SELECT 
                'bin_id' || x'09' || 'prefix' || x'09' || 'name' || x'0A' ||
                'str'    || x'09' || 'str'    || x'09' || 'str' AS record
        UNION ALL
            SELECT bin_id || x'09' || prefix || x'09' || name AS record
            FROM folders
    ),
    tsv_fs_objects AS (
        SELECT group_concat(record, x'0A') AS fs_objects
        FROM records
    )
SELECT * FROM tsv_fs_objects;
~~~

return a scalar string containing a set of records in the tab-separated value format. The last query also adds a table header.
