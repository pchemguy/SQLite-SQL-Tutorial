---
layout: default
title: Black-Boxed Queries
nav_order: 2
parent: String Manipulation
permalink: /strings/black-boxed-queries
---

Abstraction is a means to reduce coupling between different components of the system and simplify the development process. As far as RDBMS are concerned, the two main targets for abstraction are the API and SQL. Middleware libraries - a commonly used approach for abstracting both API and SQL - are not discussed here. For a basic illustration of the SQLite API abstraction, refer to this [VBA library][SQLiteCAdoReflectVBA] and a [mock VBA data manager application][ContactEditor]. The focus of this project is on SQL.

Where available, stored procedures help abstract the details of data model implementation used by the storage backend to the extent that the application may not even need to know anything about SQL. SQLite does not implement stored procedures. The next best thing is, perhaps, parameterized queries. Such queries can be included with the application as a source code SQL library and retrieved by the application code as necessary. At run time, the application pulls a query snippet from its SQL library, sends it to the database for compilation/preparation, and runs it with appropriate parameters. The only important SQL detail for the application code is the input signature of the query (the set of query parameters) and the output signature (the vector of returned columns), which is the same for stored procedures.

The previous example of the path splitting [query][DSV Query] can be converted to a parameterized query by replacing literal values in the first two WITH clause members with query parameter identifiers. For example:

~~~sql
    params(sep, path_sep) AS (VALUES (@PathVarSeparator, '\/')),
    string_data(string_id, string) AS (
        VALUES
            (1, @PathVar1),
            (2, @PathVar2)
    ),
~~~

There is one problem with this query and usage pattern, however. The *string_data* presents the primary application interface via the query parameters *@PathVar#*, where each input path variable maps to a dedicated query parameter. Because the number of these variables, which the application may need to pass to the database, is not fixed, that query cannot be fully black-boxed. To address this limitation, let us change the input specification from the "one path variable - one query parameter" approach to a single query parameter in the JSON format. For example:

~~~json
{"string_id_1": "string_1", "string_id_2": "string_2",...}
~~~

This way, a single string parameter passes an arbitrary number of input pairs (path variable and an associated ID) to the parameterized query. The number of required parameters no longer depends on the input, and the updated code can be black-boxed. The only remaining task is to add a new interface query, *json_object_data*, at the top and add code, converting *json_object_data* input to *string_data* format. The following snippet replaces the original *string_data* block:

~~~sql
	json_object_data(json_str) AS (
		VALUES
			('{' || 
			   '"1":"\\usr\\share\\man\\::bin:etc/mc:", ' ||
			   '"2":"/dev/stderr/:/dev/stdout/"' ||
			 '}')
	),	
    string_data AS (
        SELECT "key" AS string_id, "value" AS string
        FROM json_object_data AS jd, json_each(jd.json_str)
    ),
~~~

Furthermore, *json_object_data* and *string_data* can be collapsed, simplifying the code. The new body of *string_data* includes a JSON parser:

~~~sql
    string_data AS (
        SELECT "key" AS string_id, "value" AS string FROM json_each(
			'{' || 
			  '"1":"\\usr\\share\\man\\::bin:etc/mc:", ' ||
			  '"2":"/dev/stderr/:/dev/stdout/"' ||
			'}'
		)
    ),
~~~

And the final query:

~~~sql
WITH
    params(sep, path_sep) AS (VALUES (':', '\/')),
    string_data AS (
        SELECT "key" AS string_id, "value" AS string FROM json_each(
			'{' || 
			  '"1":"\\usr\\share\\man\\::bin:etc/mc:", ' ||
			  '"2":"/dev/stderr/:/dev/stdout/"' ||
			'}'
		)
    ),
    clean_strings AS (
        SELECT strs.string_id,
               replace(replace(trim(strs.string, p.sep), p.sep || p.sep, p.sep), '\', '\\') AS string
        FROM string_data AS strs, params AS p
    ),
    json_strings AS (
        SELECT strs.string_id,
               '["' || replace(strs.string, p.sep, '", "') || '"]' AS string
        FROM clean_strings AS strs, params AS p
    ),
    raw_terms AS (
        SELECT string_id, terms.id AS term_id, terms.value AS term
        FROM json_strings AS strs, json_each(strs.string) AS terms
    ),
    terms AS (
        SELECT string_id, term_id, replace(trim(term, p.path_sep), '\', '/') AS term
        FROM raw_terms, params AS p
    )
SELECT * FROM terms;
~~~

Similarly, the query can be converted to a parameterized query as follows:

~~~sql
    params(sep, path_sep) AS (VALUES (@PathVarSeparator, '\/')),
    string_data AS (
        SELECT "key" AS string_id, "value" AS string FROM json_each(@JSONPathVars)
    ),
~~~

Note how the modular nature of CTEs simplifies code modification.

<!-- References -->

[SQLiteCAdoReflectVBA]: https://pchemguy.github.io/SQLiteC-for-VBA/
[ContactEditor]: https://pchemguy.github.io/ContactEditor/
[DSV Query]: /strings/split-dsv#DSV-Query
