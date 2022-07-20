---
layout: default
title: SQL Interface - JSON Input
nav_order: 4
parent: Design Patterns
permalink: /patterns/json-sql-input
---

The JSON is a popular and robust format for passing structured information in text form. JSON libraries are broadly available in various environments, including RDBMS engines. The JSON format can serve as a kind of SQL interface for parameterized queries. When an application needs to pass a 1D vector of arbitrary length to the script, it is impossible to have a fixed parameterized script with dedicated query parameters assigned to individual values. Instead, the application packs the data into the JSON format and passes these data in a single string query parameter. The SQL script, in turn, incorporates the code for unpacking the JSON format before further processing.

JSON encoding of the passed data has several benefits. It permits having a fixed SQL script accepting an array of arbitrary length as a query parameter. Also, it establishes a relatively simple, robust, and well-defined SQL interface based on a broadly supported format. Finally, because each query parameter may cost an additional API call, this approach may also improve the overall performance of the database call. An important consideration to bear in mind related to the JSON containers is the potential side effects of data conversion between numeric and textual formats.

Consider a table *fs_objects(bin_id, prefix, name)* containing a list of file system objects, uniquely identified by their absolute paths *concat*(*prefix*, *path_sep*, *name*) and a unique *bin_id*. Suppose an application needs to pass a set of new objects for insertion into this table. Such a set is a 1D vector of arbitrary size, which may contain:

 1. **a scalar value for a single column** (e.g., absolute path).
 2. **a pair of scalar values for two columns** (e.g., id and absolute path)
 3. **a 1D array of attribute-value pairs for multiple columns** (e.g., bin_id, prefix, name)
 
For each of the three formats, we need a parameterized query accepting an arbitrary length 1D vector as input (query code and the number of query parameters must not depend on the input length). This task is perfectly suitable for the JSON format.

---

### 1. 1D vector of scalars via JSON array

**Input**

~~~json
["value1", "value2", ...]
~~~

**Query**

~~~sql
WITH
    folders AS (
        SELECT
            dirs."key" AS id,
            dirs.value AS path
        FROM
            json_each(
                '['                                            ||
                    '"C:/Winows/System32/drivers/etc/hosts",'  ||
                    '"C:/Users/Public/Desktop/pic",'           ||
					'"C:/Users/Default/Music/drum"'            ||
				']'
			) AS dirs
	)
SELECT * FROM folders;
~~~

**Output**

| id | path                                 |
|----|--------------------------------------|
| 0  | C:/Winows/System32/drivers/etc/hosts |
| 1  | C:/Users/Public/Desktop/pic          |
| 2  | C:/Users/Default/Music/drum          |

**Parameterized query**

~~~sql
WITH
    folders AS (
        SELECT dirs."key" AS id, dirs.value AS path
        FROM json_each(@Paths) AS dirs
	)
SELECT * FROM folders;
~~~

---

### 2. 1D vector of pairs via JSON object

**Input**

~~~json
{"pair1-value1": "pair1-value2", "pair2-value1": "pair2-value2", ...}
~~~

**Query**

~~~sql
WITH
    folders AS (
        SELECT
            dirs."key" AS bin_id,
            dirs.value AS path
        FROM
            json_each(
                '{'                                                   ||
                    '"239": "C:/Winows/System32/drivers/etc/hosts",'  ||
                    '"876": "C:/Users/Public/Desktop/pic",'           ||
					'"374": "C:/Users/Default/Music/drum"'            ||
				'}'
			) AS dirs
	)
SELECT * FROM folders;
~~~

**Output**

| bin_id | path                                 |
|--------|--------------------------------------|
| 239    | C:/Winows/System32/drivers/etc/hosts |
| 876    | C:/Users/Public/Desktop/pic          |
| 374    | C:/Users/Default/Music/drum          |

**Parameterized query**

~~~sql
WITH
    folders AS (
        SELECT dirs."key" AS bin_id, dirs.value AS path
        FROM json_each(@Paths) AS dirs
	)
SELECT * FROM folders;
~~~

---

### 2. 1D vector of multicolumn row values via JSON array of objects


**Input**

~~~json
[
    {"attr1": "value1_1", "attr2": "value1_2", "attr3": "value1_3"},
    {"attr1": "value2_1", "attr2": "value2_2", "attr3": "value2_3"},
    ...
]
~~~

**Query**

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
	)
SELECT * FROM folders;
~~~

**Output**

| bin_id | prefix                         | name  |
|--------|--------------------------------|-------|
| 239    | C:/Winows/System32/drivers/etc | hosts |
| 876    | C:/Users/Public/Desktop        | pic   |
| 374    | C:/Users/Default/Music         | drum  |

**Parameterized query**

~~~sql
WITH
    folders AS (
        SELECT
            json_extract(dirs.value, '$.bin_id') AS bin_id,
            json_extract(dirs.value, '$.prefix') AS prefix,
            json_extract(dirs.value, '$.name')   AS name
        FROM json_each(@Paths) AS dirs
	)
SELECT * FROM folders;
~~~

---
